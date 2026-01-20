### ts-error-F11
#### Original description

> F11 is a fault due to the lack of sequence control.
> Note that such a fault may not always occur, making  it difficult to analyze.
> Sometimes if a return value is too far from the normal value, the microsevice will recheck the correctness of the value and correct it.
> But this process does not always happen, making the result sometimes wrong but sometimes right.
>
>
>
> TrainTicket replicated fault description:
>
> In TrainTicket system, there are two microservices set the same value in the database in order cancellation process.
> Due to the lack of control like F1, the two microservices may set the value in a wrong sequence.
> However, the second microservice that set the value may recheck the value and correct the value.
> The recheck process does not always happen.
> If two microservices set the value in a wrong sequence but the recheck process does not executed, this fault occurs.
>
>
#### Original reproduction steps:

> Setup System:
>
> 1. Use docker-compose to setup the TrainTicket system.
> 2. Log in and make sure that there is at least one ticket order that fits the following:
     >    (1) The train number is start with Z or K
     >    (2) The order status is PAID
>
>
> Failure Triggering Usage Steps:
>
> 1. Log in.
> 2. Click [Flow Two - Ticket Cancel & Ticket Change].
> 3. Click [Refresh Orders].
> 4. Select the order mentioned above and click [Cancel Order].
> 5. Click [Confirm Cancel].
> 6. You will get result of cancel. If you get [SUCCESS] means the fault do not occur.
     >    If you get [Error] alert, that means the fault occurs, and you will see the exception logs on the server console.

Mostly works as expected. The involved microservices are ts-cancel-service, ts-inside-payment-service
and ts-order-other-service.
The erroneous workflow consists of the following steps (according to original code comments). All actions start from
ts-cancel-service.CancelServiceImpl#CancelOrder():
1. ts-cancel-service persists the order status to "CANCELING" with an async call to ts-order-other-service in line 130. This call
   has a 40% chance to have a delay of four seconds.
2. Line 133 ts-cancel-service draw back / refund the money with an async call to ts-inside-payment-service. This call has a fixed
   delay of two seconds. In ts-inside-payment-service.InsidePaymentServiceImpl#drawBack() the order status is set and
   persisted to "CANCEL". Starting from line 272 in InsidePaymentServiceImpl#drawBack() there also is some code which looks like the recheck functionality mentioned
   in the original description. However, this functionality is never executed because a boolean variable prevents it. In the
   context where this call is made, it always returns "true". This variable has to be set with a call to an endpoint defined in InsidePaymentController (openRecheck / closeRecheck)
3. In ts-cancel-service get and check the status of the order. If it is "CANCELING", then the order status is wrong, else
   a correct order status is assumed. If the order status is wrong, the method throws an exception, else it returns normally.

As stated in the original description, it is correct that the cancel call will fail sometimes. However, it is not correct
that the second microservice involved (ts-inside-payment-service) will sometimes try to correct the order status.
