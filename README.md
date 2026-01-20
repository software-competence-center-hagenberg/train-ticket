### ts-error-F2
#### Original fault description
> Industrial fault description:
>
> F2 is a fault that multiple messages are displayed in wrong order.
> Sometimes we send many requests at a short time.
> However, these requests may returns at a different order with the origin.
> If we display the information without checking the order of these requests, wrong information may be shown on the screen.
> Then this kind of fault occurs.
>
>
> TrainTicket replicated fault description:
>
> In TrainTicket system, every request for ticket-reservation has a random delay to simulate the unstable network condition.
> If a user send multiple requests for ticket-reservation at a short time, the order of the result may be wrong due to the random delay.
> Then this fault occurs.
>
>
> 1. Login
> 2. Search for a ticket using the following info:
     > Starting Place: Shang Hai
     > Terminal Place: Su Zhou
     > Date: After today
     > Train Type: GaoTie DongChe
> 3. Click [Booking]
> 4. Click [Refresh Contacts]
     > Select one of the Contacts and click [Select]
> 5. Click [Confirm Ticket]
     > You may try some more times if you get a "Success". If the fault occurs, you will get an [Error] and you
     > will see the exception logs in console.
>
> ### Tip
> In order to simplify the process of log collection, we add a service name click-twice.
> If you click [Confirm Ticket], a request will be sent to click-twice, and this service will send two
> request to ticket-preserve asynchronous. In some cases the fault will occurs.
>
> ### How the fault occurs
> 1. In [ts-preserve-service] -> [PreserController], a var named [statusBean] will save the unfinished request.
> 2. In [ts-preserve-service] -> [PreserviceImpl] -> [preserve], a random delay is added at the beginning of this method.
> 3. For a user who send 2 requests to [ts-preservice-service] in a short time, the 2nd request may be finished early before the first one due to the random delay.
     >    In such situation, the system will found that this user has a non-Finished request which should be finished before the 2nd request.
     >    Then the exception will be thrown.

#### Notes
The reproduction works as described. However, the original implementation of `click-twice` does not send back the result the front end
expects. Therefore, the ticket booking workflow can not be continued after a successful confirmation. To allow continuing
the process, we adapted the return type of the controller `ClickController` to the correct type.

This branch introduces a new object `OrderTicketsInfoPlus` which is needed in some endpoints. This new object consists of
fields for `loginId`, `loginToken` and an `OrderTicketsInfo` object. It is solely used in ts-click-twice and ts-preserve-service
to pass on the loginId. It is not quite clear, why this new object is really needed, because in the 'error-free' version,
`loginId` and `loginToken` are passed as cookies.

There seems to be some mismatch between description and actual behaviour. The description states that there is a problem with
displaying the reservation id. Example scenario:
1. Reservations A and B are issued with little to no delay in this order.
2. Due to random delay during processing the reservations, B finishes before A does.
3. In the frontend the reservation id of B is displayed although the id of A should be displayed.

Actual behaviour:
1. Reservations A and B are issued with little to no delay in this order.
2. Due to random delay during processing the reservations, B finishes before A does.
3. Execution of A is aborted.
