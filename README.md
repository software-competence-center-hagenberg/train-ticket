### ts-error-F6

#### Original fault description
> A service is slowing down and returns error finally.
> Endless recursive requests of a microservice are caused by SQL errors of another dependent microservice.
>
>
> ## TrainTicket replicated fault description:
>
> In out train ticket system, we replicate this fault in voucher service.
> The SQL arg in voucher service is something wrong, make the request fail.
> When the request is fail, the host service will send another request.
> After many retries, we will receive a timeout error finally.
>
>
> ## Failure Triggering Usage Steps:
> The fault occurs when trying to cancel a ticket(normal ticket which is paid and not collected).
>
> 1. Log in.
> 2. Click [Flow Two - Ticket Cancel & Ticket Change].
> 3. Click [Refresh Orders].
> 4. Select the order mentioned above and click [Cancel Order].
> 5. Click [Confirm Cancel].
> 6. You will receive a timeout.
>

#### Notes
The ticket has to have a trip id starting with Z, K or T (e.g. a trip from Shang Hai to Nan Jing).
`ts-cancel-service` will retry for 10 times before throwing an exception. Note that the status of the canceled ticket
is not updated in the UI. Hence, one has to click `Refresh Orders` again. `ts-voucher-service` does not produce any logs
during this process.

A faulty SQL DELETE statement in `ts-voucher-service` (the query references the column 'voucherId', the correct name is 'voucher_id') causes the request to ts-voucher-service to fail constantly (Note: although the called POST endpoint is named 'getVoucher' it actually is a DELETE operation). When the request fails, which it always does, `ts-cancel-service` retries the request after a five second sleep for ten times. After that it throws a generic exception. It is debateable if the usage of the term 'timeout' is appropriate here, because there is no timeout involved.
