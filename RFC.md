Glossary:

These are terms that I think deserves an up-front definition in the spec:

    - `signal`
    - `signaller`
    - `emit` / `send` <-- change spec to use one of these
    - `fail` / `failure` / `failing`
    - `demand`
    - `item`/`data`/`element` <-- change spec to use one of these
    - `upstream` / `downstream`
    - `return normally`
    - `terminal state` / `end-of-stream` <-- change spec to use one of these
    - `responsivity` / `non-obstructing` <-- change spec to use one of these
    - `call`/`invoke` <-- change spec to use one of these

Publisher: 1

§1
`The total number of onNext signals sent by a Publisher to a Subscriber MUST be less than or equal to the total number of elements requested by that Subscriber´s Subscription at all times.`

Comments:
Publishers cannot emit more elements than Subscribers have requested. There's an implicit, but important, consequence to this rule:
Since demand can only be fulfilled after it has been received, there's a
happens-before relationship between requesting elements and emitting elements.

§2
`A Publisher MAY signal less (FIXME: FEWER) onNext than requested and terminate the Subscription by calling onComplete or onError.`

Comments:
Publisher cannot guarantee that it will be able to produce the number of elements requested; it simply might not be able to produce them all; it may be in a failed state; it may be empty or otherwise already completed.

§3
`onSubscribe, onNext, onError and onComplete signaled to a Subscriber MUST be signaled sequentially (no concurrent notifications).`

Comments:
This means that `external synchronization` must be employed if the Publisher
intends to send signals from multiple/different threads.

§4
`If a Publisher fails it MUST signal an onError.`

Comments:
The intent of this rule is to make it clear that a Publisher is responsible for notifying its Subscribers if it detects that it cannot proceed—Subscribers
must be given a chance to clean up resources or otherwise deal with `upstream` failures.

§5
`If a Publisher terminates successfully (finite stream) it MUST signal an onComplete.`

Comments:
The intent of this rule is to make it clear that a Publisher is responsible for notifying its Subscribers that it has reached `end-of-stream`—Subscribers
 can then act on this information; clean up resources, etc.

§6
`If a Publisher signals either onError or onComplete on a Subscriber, that Subscriber’s Subscription MUST be considered cancelled.`

Comments:
The intent of this rule is to make sure that a Subscription is treated the same no matter if it was cancelled, the upstream signalled onError or onComplete.

§7
`Once a terminal state has been signaled (onError, onComplete) it is REQUIRED that no further signals occur.`

Comments:
The intent of this rule is to make sure that onError and onComplete are the final states of an interaction between Publisher and Subscriber.

§8
`If a Subscription is cancelled its Subscriber MUST eventually stop being signaled.`

Comments:
The intent of this rule is to make sure that Publishers respect a Subscriber's
request to cancel a Subscription when Subscription.cancel() has been invoked.

§9
`Publisher.subscribe MUST call onSubscribe on the provided Subscriber prior to any other signals to that Subscriber and MUST return normally, except when the provided Subscriber is null in which case it MUST throw a java.lang.NullPointerException to the caller, for all other situations the only legal way to signal failure (or reject the Subscriber) is by calling onError (after calling onSubscribe).``

Comments:
The intent of this rule is to make sure that `onSubscribe` is always signalled
before any of the other signals, so that initialization logic can be executed by the Subscriber when the signal is received.
If the supplied `Subsccriber` is `null`, there is nowhere else to signal this but to the caller, which means a `java.lang.NullPointerException` must be thrown.

Examples of possible situations: A stateful Publisher can be overwhelmed, bounded by a finite number of underlying resources, exhausted, shut-down or in a failed state.

`Return normally` means returning control to the caller. [FIXME: glossary?]


§10
`Publisher.subscribe MAY be called as many times as wanted but MUST be with a different Subscriber each time [see 2.12].`

Comments:
The intent of this rule is to have callers of `subscribe` be aware that a generic Publisher and a generic Subscriber cannot be assumed to support being attached multiple times.

Furthermore, it also mandates that the semantics of `subscribe` must be upheld no matter how many times it is invoked.

§11
`A Publisher MAY support multiple Subscribers and decides whether each Subscription is unicast or multicast.`

Comments:
The intent of this rule is to give Publisher implementations the flexibility to decide how many, if any, Subsribers they will support, and how elements are going to be distributed.






Subscriber: 2

§1
`A Subscriber MUST signal demand via Subscription.request(long n) to receive onNext signals.`

Comments:
The intent of this rule is to establish that it is the responsibility of the Subscriber to signal when, and how many, elements it is able and willing to receive.

§2
`If a Subscriber suspects that its processing of signals will negatively impact its Publisher's responsivity, it is RECOMMENDED that it asynchronously dispatches its signals.`

Comments:
The intent of this rule is that a Subscriber should not impede the progress of the Publisher from an execution point-of-view. In other words, the Subscriber should not starve the Publisher from CPU cycles

§3
`Subscriber.onComplete() and Subscriber.onError(Throwable t) MUST NOT call any methods on the Subscription or the Publisher.` 

Comments:
The intent of this rule is to prevent cycles and race-conditions—between Publisher, Subsription and Subscriber—during the processing of completion signals.

§4
`Subscriber.onComplete() and Subscriber.onError(Throwable t) MUST consider the Subscription cancelled after having received the signal.`

Comments:
The intent of this rule is to make sure that Subscribers respect a Publisher's
terminal state signals. A Subscription is simply not valid anymore after an onComplete or onError signal has been received.


§5
`A Subscriber MUST call Subscription.cancel() on the given Subscription after an onSubscribe signal if it already has an active Subscription.`

Comments:
The intent of this rule is to prevent that two, or more, separate Publishers from thinking that they can communicate with the same Subscriber. Enforcing this rule means that resource leaks are prevented since extra Subscriptions will be cancelled.

§6
`A Subscriber MUST call Subscription.cancel() if it is no longer valid to the Publisher without the Publisher having signaled onError or onComplete.`

Comments:
The intent of this rule is to establish that Subsribers cannot just throw Subscriptions away when they are no longer needed, they have to call `cancel` so that resources held by that Subscription can be safely, and timely, reclaimed.

§7
`A Subscriber MUST ensure that all calls on its Subscription take place from the same thread or provide for respective external synchronization.`

Comments:
The intent of this rule is to establish that external synchronization must be added if a Subsriber will be using a Subscription concurrently by two or more threads.

§8
`A Subscriber MUST be prepared to receive one or more onNext signals after having called Subscription.cancel() if there are still requested elements pending [see 3.12]. Subscription.cancel() does not guarantee to perform the underlying cleaning operations immediately.`

Comments:
The intent of this rule is to highlight that there may be a delay between calling `cancel` the Publisher seeing that.

§9
`A Subscriber MUST be prepared to receive an onComplete signal with or without a preceding Subscription.request(long n) call.`

Comments:
The intent of this rule is to establish that completion is unrelated to the demand flow—this allows for streams which close early, and does not need to "poll for completion".

§10
`A Subscriber MUST be prepared to receive an onError signal with or without a preceding Subscription.request(long n) call.`

Comments:
The intent of this rule is to establish that Publisher failures may be completely unrelated to signalled demand. This means that Subscribers do not need to poll to find out if the upstream will not be able to fulfill its requests.

§11
`A Subscriber MUST make sure that all calls on its onXXX methods happen-before [1] the processing of the respective signals. I.e. the Subscriber must take care of properly publishing the signal to its processing logic.`

Comments:
The intent of this rule is to establish that it is the responsibility of the Subsriber implementation to make sure that asynchronous processing of its signals are thread safe.

See JMM definition of Happen-Before in section 17.4.5. on http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html

§12
`Subscriber.onSubscribe MUST be called at most once for a given Subscriber (based on object equality).`

Comments:
The intent of this rule is to establish that it is not to be considered intended behavior that the same Subscriber is concurrently subscribed to several Publishers.

§13
`Calling onSubscribe, onNext, onError or onComplete MUST return normally except when any provided parameter is null in which case it MUST throw a java.lang.NullPointerException to the caller, for all other situations the only legal way for a Subscriber to signal failure is by cancelling its Subscription. In the case that this rule is violated, any associated Subscription to the Subscriber MUST be considered as cancelled, and the caller MUST raise this error condition in a fashion that is adequate for the runtime environment.`

Comments:

The intent of this rule is to establish the semantics for the methods of Subscriber and what the Publisher is allowed to do in which case this rule is violated. By "raising this error condition in a fashion that is adequate for the runtime environment" could mean logging the error or otherwise make someone or something aware of the situation, as the error cannot be sent to the faulty Subscriber.

[FIXME: remove as it is baked into comment section]
[1] : See JMM definition of Happen-Before in section 17.4.5. on http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html



Subscription: 3

§1
`Subscription.request and Subscription.cancel MUST only be called inside of its Subscriber context (FIXME: add [see 2.7]). (FIXME: move this into the comments section)A Subscription represents the unique relationship between a Subscriber and a Publisher [see 2.12].`

Comments:
The intent of this rule is to establish that a Subscription represents the unique relationship between a Subscriber and a Publisher [see 2.12].
The Subscriber is in control over when elements are requested and when more elements are no longer needed.

§2
`The Subscription MUST allow the Subscriber to call Subscription.request synchronously from within onNext or onSubscribe.`

Comments:
The intent of this rule is to make it clear that implementations of `request` must be reentrant, to avoid stack overflows in the case of mutual recursion between `request` and `onNext` (and eventually `onComplete` / `onError`). This implies that Publishers can be `synchronous`, i.e. emitting elements on the thread which invokes `request`.

§3
`Subscription.request MUST place an upper bound on possible synchronous recursion between Publisher and Subscriber[1].` (FIXME: remove [1])

Comments:
The intent of this rule is to complement [see 3:2] by placing an upper limit on the mutual recursion between `request` and `onNext` (and eventually `onComplete` / `onError`). Implementations are RECOMMENDED to limit this mutual recursion to a depth of `1` (ONE)—for the sake of conserving stack space.

An example for undesirable synchronous, open recursion would be Subscriber.onNext -> Subscription.request -> Subscriber.onNext -> …, as it very quickly would result in blowing the calling Thread´s stack.

§4
`Subscription.request SHOULD respect the responsivity of its caller by returning in a timely manner[2].` (FIXME: remove [2])

Comments:
The intent of this rule is to establish that `request` is intended to be a non-obstructing method, and should be as quick to execute as possible on the calling thread, so avoid heavy computations and other things that would stall the caller´s thread of execution.

§5
`Subscription.cancel MUST respect the responsivity of its caller by returning in a timely manner[2], (FIXME: remove [2]) MUST be idempotent and MUST be thread-safe (FIXME: remove thread-safe part, as it is covered by 3.1 and 2.7).`

Comments:
The intent of this rule is to establish that `cancel` is intended to be a non-obstructing method, and should be as quick to execute as possible on the calling thread, so avoid heavy computations and other things that would stall the caller´s thread of execution. Furthermore, it is also important that it is possible to invoke it multiple times without any adverse effects.

§6
`After the Subscription is cancelled, additional Subscription.request(long n) MUST be NOPs.`

Comments:
The intent of this rule is to establish a causal relationship between cancellation of a subscription and the subsequent non-operation of requesting more elements.

§7
`After the Subscription is cancelled, additional Subscription.cancel() MUST be NOPs.`

Comments:
[FIXME: reword or fold into 3:5? Or edit 3:5 to not mention idempotency?]

§8
`While the Subscription is not cancelled, Subscription.request(long n) MUST register the given number of additional elements to be produced to the respective subscriber.`

Comments:
The intent of this rule is to make sure that `request`-ing is an additive operation, as well as ensuring that a request for elements is delivered to the Publisher.

§9
`While the Subscription is not cancelled, Subscription.request(long n) MUST signal onError with a java.lang.IllegalArgumentException if the argument is <= 0. The cause message MUST include a reference to this rule and/or quote the full rule.`

Comments:
The intent of this rule is to prevent faulty implementations to proceed operation without any exceptions being raised. Requesting a negative or 0 number of elements, since requestion is additive, most likely to be the result of an erronous calculation on the behalf of the Subscriber.

§10
`While the Subscription is not cancelled, Subscription.request(long n) MAY synchronously call onNext on this (or other) subscriber(s).`

Comments:
The intent of this rule is to establish that it is allowed to create synchronou Publishers, i.e. Publishers who execute their logic on the calling thread.

§11
`While the Subscription is not cancelled, Subscription.request(long n) MAY synchronously call onComplete or onError on this (or other) subscriber(s).`

Comments:
The intent of this rule is to establish that it is allowed to create synchronou Publishers, i.e. Publishers who execute their logic on the calling thread.

§12
`While the Subscription is not cancelled, Subscription.cancel() MUST request the Publisher to eventually stop signaling its Subscriber. The operation is NOT REQUIRED to affect the Subscription immediately.`

Comments:
The intent of this rule is to establish that the desire to cancel a Subscriptiuon is eventually respected by the Publisher, acknowledging that it may take some time before the signal is received.

§13
`While the Subscription is not cancelled, Subscription.cancel() MUST request the Publisher to eventually drop any references to the corresponding subscriber. [FIXME: move the following sentence to the comments section for this rule]Re-subscribing with the same Subscriber object is discouraged [see 2.12], but this specification does not mandate that it is disallowed since that would mean having to store previously cancelled subscriptions indefinitely.`

Comments:
The intent of this rule is to make sure that Subsribers can be properly garbage-collected after their subsription no longer being valid.
Re-subscribing with the same Subscriber object is discouraged [see 2.12], but this specification does not mandate that it is disallowed since that would mean having to store previously cancelled subscriptions indefinitely.

§14
`While the Subscription is not cancelled, calling Subscription.cancel MAY cause the Publisher, if stateful, to transition into the shut-down state if no other Subscription exists at this point [see 1.9].`

Comments:
The intent of this rule is to allow for Publishers to not accept new Subscribers in response to a cancellation signal. [FIXME: is this overly restrictive? Shouldn't a Publisher be allowed to deny new Subscribers at its own discretion?]

§15
`Calling Subscription.cancel MUST return normally. The only legal way to signal failure to a Subscriber is via the onError method.`

Comments:
The intent of this rule is to disallow implementations to throw exceptions in response to `cancel` being invoked, instead directing implementations to signal exceptions through the Subscriber's `onError`.

§16
`Calling Subscription.request MUST return normally. The only legal way to signal failure to a Subscriber is via the onError method.`

Comments:
The intent of this rule is to disallow implementations to throw exceptions in response to `cancel` being invoked, instead directing implementations to signal exceptions through the Subscriber's `onError`.

§17
`A Subscription MUST support an unbounded number of calls to request and MUST support a demand (sum requested - sum delivered) up to 2^63-1 (java.lang.Long.MAX_VALUE). A demand equal or greater than 2^63-1 (java.lang.Long.MAX_VALUE) MAY be considered by the Publisher as “effectively unbounded”[3].` (FIXME: remove [3])

Comments:
The intent of this rule is to establish that the Subscriber can request an unbounded number of elements, in any increment above 0 [see 3:9], in any number of invocations of `request`.
As it is not feasibly reachable with current or foreseen hardware within a reasonable amount of time (1 element per nanosecond would take 292 years) to fulfill a demand of 2^63-1, it is allowed for a Publisher to stop tracking demand beyond this point.

(FIXME: remove these as they have been folded into the comments)

[1] : An example for undesirable synchronous, open recursion would be Subscriber.onNext -> Subscription.request -> Subscriber.onNext -> …, as it very quickly would result in blowing the calling Thread´s stack.

[2] : Avoid heavy computations and other things that would stall the caller´s thread of execution

[3] : As it is not feasibly reachable with current or foreseen hardware within a reasonable amount of time (1 element per nanosecond would take 292 years) to fulfill a demand of 2^63-1, it is allowed for a Publisher to stop tracking demand beyond this point.



Processor: 4

§1
`A Processor represents a processing stage—which is both a Subscriber and a Publisher and MUST obey the contracts of both.`

Comments:
The intent of this rule is to establish that Processors behave, and are bound by, both the Publisher and Subscriber specifications.

§2
`A Processor MAY choose to recover an onError signal. If it chooses to do so, it MUST consider the Subscription cancelled, otherwise it MUST propagate the onError signal to its Subscribers immediately.` (FIXME: should we expressedly allow for `onComplete` as well?)

Comments:
The intent of this rule is to inform that it's possible for implementations to be more than simple transformations.

