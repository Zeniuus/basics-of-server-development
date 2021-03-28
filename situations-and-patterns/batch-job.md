Batch job은 비즈니스 로직을 작성할 때 빈번하게 사용되는 패턴이다. Batch job은 수많은 작업을 한 번에 처리하는데, 이 때문에 batch job을 작성할 때는 단순히 API를 작성할 때보다 더 세심한 주의를 기울여야 한다. 어떤 점을 주의해야 하는지 알아보기 위해, 매년 개인정보 이용내역 안내 메일을 발송하는 간단한 batch job 예시를 가져왔다. (Spring + Kotlin 스택에서 작성되었다)

```kotlin
@Transactional(isolation = Isolation.SERIALIZABLLE)
fun process() {
    val users = userRepository.findPrivacyUsageMailSendingTargets()
    users.forEach { user ->
        sendPrivacyUsageMail(user)
    }
}

private fun sendPrivacyUsageMail(user: User) {
    emailService.sendPrivacyUsageMail(user)
    user.privacyUsageMailSentAt = clock.instant()
    userRepository.save(user)
}
```

겉보기엔 멀쩡해 보이지만, 위 batch job 코드에는 많은 문제가 숨어 있다. 지금부터 문제를 하나씩 개선해보자.

## 하나의 실패가 전체 실패로 이어지지 않도록 하기

첫 번째 문제로, 한 유저에게 메일을 발송하다가 exception이 던져지면 그 이후의 모든 유저에게 메일 발송이 실패한다. 예를 들어 첫 번째 유저에게 메일을 전송하다가 네트워크 오류 때문에 exception이 던져졌다고 해보자. 그러면 해당 exception은 `process()` 함수 전체를 중단시킬 것이다.

이를 방지하는 방법은 아주 간단하다. `users`를 순회하면서 적용하는 로직에 try-catch를 걸면 된다.

```kotlin
@Transactional(isolation = Isolation.SERIALIZABLLE)
fun process() {
    val users = userRepository.findPrivacyUsageMailSendingTargets()
    users.forEach { user ->
        try {
            sendPrivacyUsageMail(user)
        } catch (t: Throwable) {
            logger.error(t.message, t)
        }
    }
}
```

## 트랜잭션 경계 잘 설정하기

위 batch job의 로직을 잘 보면 메일을 발송한 이후 메일 발송 시각을 DB에 적는다. 이 때문에 user에 X lock이 잡히게 된다. 하지만 지금 트랜잭션 경계를 보면 `process()` 함수 전체에 잡혀 있다. 이런 긴 batch job이 X lock을 잡으면 batch job이 끝날 때까지 user를 변경하는 모든 쿼리가 막히기 때문에 치명적인 장애로 이어질 수 있다.

이를 방지하려면 트랜잭션의 범위를 잘 분리하면 된다. 바로 아래와 같이 말이다.

```kotlin
fun process() {
    val users = transactionTemplate.execute {
        userRepository.findPrivacyUsageMailSendingTargets()
    }!!
    users.forEach { user ->
        try {
            serializableTransactionTemplate.execute {
                sendPrivacyUsageMail(user)
            }
        } catch (t: Throwable) {
            logger.error(t.message, t)
        }
    }
}
```

여기서 주의할 점은, serializable 트랜잭션을 try-catch 안에 잡는 것이다. serializable 트랜잭션은 데드락으로 인해 exception이 던져질 가능성이 높기 때문이다. 만약 serializable 트랜잭션을 try-catch 바깥에 잡으면 이런 exception을 잡지 못할 것이고, 앞서 보았던 **하나의 실패가 전체 실패로 이어지지 않도록 하기**에서 보았던 문제가 재발할 것이다.

## 동시 수정에 안전하게 만들기 (혹은 멱등적으로 만들기)

위의 개선으로 인해 발생한 새로운 문제가 있다. 예를 들어, batch job이 아닌 다른 곳에서 `user.privacyUsageMailSentAt`을 갱신하는 로직이 있다고 해보자. 만약 해당 로직이 `users`를 가져오는 트랜잭션과 실제로 메일을 발송하는 트랜잭션 사이에 실행된다면, 메일이 중복 발송되는 일이 발생할 수도 있다.

이런 문제가 발생하는 근본적인 이유는 `user`를 조회해오는 시점과 메일을 발송하는 로직이 하나의 트랜잭션으로 묶이지 않았기 때문이다. 원래대로라면 하나의 serializable 트랜잭션에서 실행되어야 할 두 개의 작업이 batch job의 특성 때문에 서로 다른 트랜잭션에서 실행돼서 도중에 `user`의 상태가 변할 수 있게 된 것이다.

이를 방지하려면 유저에게 메일을 발송하는 트랜잭션에서 최신 상태의 유저를 다시 조회해와야 한다. 즉, 두 개의 작업을 하나의 serializable 트랜잭션에서 실행하면 된다.

```kotlin
fun process() {
    val users = transactionTemplate.execute {
        userRepository.findPrivacyUsageMailSendingTargets()
    }!!
    users.forEach { oldUser ->
        try {
            serializableTransactionTemplate.execute {
                val user = userRepository.getOne(oldUser.id)!!
                if (shouldSendPrivacyUsageMail(user)) {
                    sendPrivacyUsageMail(user)
                }
            }
        } catch (t: Throwable) {
            logger.error(t.message, t)
        }
    }
}

private fun shouldSendPrivacyUsageMail(user: User): Boolean {
    return user.privacyUsageMailSentAt == null ||
        user.privacyUsageMaillSentAt + Duration.days(364) < clock.instant()
}
```

이는 결국, 메일 발송 로직을 멱등적으로 만들어야 한다는 것과 같다. 멱등성이란, 같은 로직이 두 번 이상 트리거 된 경우에도 한 번만 처리되는 성질이다. 위에서 언급한 이유 말고도 다양한 원인으로 인해 동일한 `user` 상태에 대해 메일 발송 로직이 두 번 이상 트리거될 수 있다. 예를 들어, 같은 batch job이 두 번 이상 실행된다면? 이런 모든 상황에 대해 메일 발송을 안전하게 수행할 수 있는 유일한 방법은, 메일 발송 로직을 멱등적으로 만드는 것뿐이다.

## Timeout 방지하기

어플리케이션의 내부 스케쥴러가 아니라 외부 cronjob과 curl 커맨드에 의해 실행되는 batch job을 생각해보자. 해당 batch job은 curl 커맨드의 request timeout과 웹서버에 등록된 response timeout의 제한을 받는다. 예를 들어 batch job 실행에는 수 분이 걸리는데, request timeout이 30초로 설정되어 있다고 해보자. 이러면 batch job 실행 자체는 성공하지만 curl 커맨드는 timeout으로 요청이 실패했다고 판단할 것이다. 더 심각한 건 response timeout이 30초로 설정되어 있는 경우인데, 이러면 batch job 실행 자체가 30초 이후에 죽어버린다.

이런 문제를 해결하기 위해 batch job을 실행시키기 위한 thread pool을 별도로 둘 수 있다.

```kotlin
private val executor = Executors.newSingleThreadExecutor()

fun process() {
    executor.submit {
        val users = transactionTemplate.execute {
            userRepository.findPrivacyUsageMailSendingTargets()
        }!!
        users.forEach { oldUser ->
            try {
                serializableTransactionTemplate.execute {
                    val user = userRepository.getOne(oldUser.id)!!
                    if (shouldSendPrivacyUsageMail(user)) {
                        sendPrivacyUsageMail(user)
                    }
                }
            } catch (t: Throwable) {
                logger.error(t.message, t)
            }
        }
    }
}
```

이제 `process()` 함수는 `executor`에게 task를 던지고 바로 종료되므로, request timeout / response timeout으로부터 안전하다.

## 로드 분산시키기

해당 기능을 배포한 뒤 batch job이 처음 실행될 경우, 상당히 많은 유저가 메일 발송 타겟이 될 것이다. 이 경우 두 가지가 문제가 된다.

- 메일 발송 API를 너무 높은 빈도로 호출하게 된다. 잘못하면 메일 발송 서버가 우리 서버의 IP를 차단할 수도 있다.
- 너무 많은 유저를 메모리에 들고 와서 메모리가 터질 수도 있다. 예를 들어 대상인 유저가 100만명이고 각 유저 객체가 100Byte라고 하면 약 100MB 정도의 메모리를 차지하게 된다. 100MB 정도면 그렇게 크지 않다고 할 수 있지만, 이런 batch job이 여러개가 되면 문제가 될 수 있다.

우선 메일 발송 API 호출 빈도 문제부터 해결해보자. 이는 단순히 발송 로직에 rate limiter를 걸면 해결된다. 아래 코드는 메일 발송 API를 최대 초당 10회까지만 호출하도록 방지한다.

```kotlin
private val rateLimiter = RateLimiter.create(10.0)

fun process() {
    executor.submit {
        val users = transactionTemplate.execute {
            userRepository.findPrivacyUsageMailSendingTargets()
        }!!
        users.forEach { oldUser ->
            rateLimiter.acquire()
            try {
                serializableTransactionTemplate.execute {
                    val user = userRepository.getOne(oldUser.id)!!
                    if (shouldSendPrivacyUsageMail(user)) {
                        sendPrivacyUsageMail(user)
                    }
                }
            } catch (t: Throwable) {
                logger.error(t.message, t)
            }
        }
    }
}
```

여기서 주의할 점은 `rateLimiter.acquire()`을 트랜잭션 바깥에서 호출한다는 점이다. 만약 트랜잭션을 열고 `rateLimiter.acquire()`를 하면 의도치 않게 트랜잭션을 오래 잡아 문제가 발생할 수도 있다.

이제 두 번째 문제인 메모리 문제를 해결해보자. 이는 유저를 단계별로 가져와서 처리하는 방식으로 해결할 수 있다. 유저가 점점 많아져서 메모리 pressure가 커지면 GC가 이미 메일을 발송한 유저를 삭제할 것이다.

```kotlin
fun process() {
    executor.submit {
        var page = transactionTemplate.execute {
            userRepository.findPrivacyUsageMailSendingTargets(limit = 1000, page = 1)
        }!!
        while (page.content.isNotEmpty()) {
            val users = page.content
            users.forEach { oldUser ->
                rateLimiter.acquire()
                try {
                    serializableTransactionTemplate.execute {
                        val user = userRepository.getOne(oldUser.id)!!
                        if (shouldSendPrivacyUsageMail(user)) {
                            sendPrivacyUsageMail(user)
                        }
                    }
                } catch (t: Throwable) {
                    logger.error(t.message, t)
                }
            }

            page = transactionTemplate.execute {
                userRepository.findPrivacyUsageMailSendingTargets(limit = 1000, page = page.currentPage + 1)
            }!!
        }
    }
}
```

## 요약  : 최종 코드

더 이상 큰 문제가 없어 보인다. 이제 처음 작성한 batch job 코드와 개선된 최종 결과물을 비교해보자.

```kotlin
@Transactional(isolation = Isolation.SERIALIZABLLE)
fun process() {
    val users = userRepository.findPrivacyUsageMailSendingTargets()
    users.forEach { user ->
        sendPrivacyUsageMail(user)
    }
}

private fun sendPrivacyUsageMail(user: User) {
    emailService.sendPrivacyUsageMail(user)
    user.privacyUsageMailSentAt = clock.instant()
    userRepository.save(user)
}
```

```kotlin
private val executor = Executors.newSingleThreadExecutor()
private val rateLimiter = RateLimiter.create(10.0)

fun process() {
    executor.submit {
        var page = transactionTemplate.execute {
            userRepository.findPrivacyUsageMailSendingTargets(limit = 1000, page = 1)
        }!!
        while (page.content.isNotEmpty()) {
            val users = page.content
            users.forEach { oldUser ->
                rateLimiter.acquire()
                try {
                    serializableTransactionTemplate.execute {
                        val user = userRepository.getOne(oldUser.id)!!
                        if (shouldSendPrivacyUsageMail(user)) {
                            sendPrivacyUsageMail(user)
                        }
                    }
                } catch (t: Throwable) {
                    logger.error(t.message, t)
                }
            }

            page = transactionTemplate.execute {
                userRepository.findPrivacyUsageMailSendingTargets(limit = 1000, page = page.currentPage + 1)
            }!!
        }
    }
}

private fun shouldSendPrivacyUsageMail(user: User): Boolean {
    return user.privacyUsageMailSentAt == null ||
        user.privacyUsageMaillSentAt + Duration.days(364) < clock.instant()
}

private fun sendPrivacyUsageMail(user: User) {
    emailService.sendPrivacyUsageMail(user)
    user.privacyUsageMailSentAt = clock.instant()
    userRepository.save(user)
}
```
