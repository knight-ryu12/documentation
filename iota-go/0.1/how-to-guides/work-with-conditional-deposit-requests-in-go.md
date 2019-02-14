# Working with conditional deposit requests

**Accounts in client libraries use the conditional deposit requests (CDRs) when communicating with depositors.**

CDRs ensure correct input selection when you send a transfer. This means that every deposit address generated by the account (with the exception of remainder addresses), includes a timestamp up to which the address is considered valid. If the current time is after that timestamp, then the address is automatically considered in input selection.

The principle behind CDRs is:

1. You generate a CDR with a specific timeout (and optionally an amount).
2. You communicate the CDR to a depositor.
3. Based on the address' timeout, the depositor assesses whether to send a transaction or not. The depositor needs to take into consideration whether he or she is able to fullfil the transaction in the given time frame. Based on network conditions, for example.
4. If the depositor isn't able to fullfil the transaction within the time frame of the CDR's timeout, he or she should request a new CDR. 

A CDR object looks like this:
```javascript
{
    // address + checksum, must be 90 trytes.
    "address": "SZEQPTJBDSYXWAUVTV...",
    // the time at which this CDR times out.
    "timeout_at": "2019-01-30T18:23:12.969227153+01:00",
    // (optional) whether multiple deposits are expected.
    "multi_use": boolean,
    // (optional) the expected amount to be deposited.
    "expected_amount": number
}
```

This object can be translated into a medium works in a given context. For example,45 on a website, a CDR can be displayed as a magnet-link. A CDR in the magnet-link format is defined as:
`iota://MBREWACWIPRFJRDYYHAAME…AMOIDZCYKW/?t=1548337187&m=true&am=0`

The account library provides utility functions to convert serialized forms of a CDR into an object.

<dl><dt>Important</dt><dd>You should never use addresses which are not in the CDR format. We recommend that you ask the receiver to supply a CDR.</dd></dl>

|  **CDR optional parameter definition** | **Behavior**
| :----------| :----------|
|Only `timeout_at` is supplied |The CDR will be used in input selection as soon as *ANY* funds reside on the given address.|
|`timeout_at` and `multi_use` are defined |The CDR will be used in input selection as the `timeout_at` parameter is fulfilled, regardless of how many deposits were made on the CDR. |
|`timeout_at`, `multi_use`, and `amount` are supplied |**RECOMMENDED:** The CDR will be used in input selection as soon as the expected amount (or more in case the last transaction that fullfils the amount also exceeds it) resides on the address. |


### Conditional deposit requests FAQ

#### What lifetime should I define for my deposit address?

This depends on how fast you expect the depositor to deposit. If you are in direct contact with the depositor and both parties are waiting eagerly to settle the transaction, a low timeout can be used. For example, 2 hours. If you are unsure how long it will take the depositor to deposit, increase the lifetime.

**Specifying amounts** - If you know the expected deposit amount upfront, which is mostly the case when requesting direct deposits for a specific scenario, you should also define the `expected amount`. This way, the address will be used in input selection as soon as it has the expected funds instead of waiting for the lifetime to end.

It does not matter whether you set the lifetime of your CDR too low, as the depositor's wallet or application should verify the CDR and assess whether given the current network conditions, the transaction would arrive in time.

<dl><dt>Important: CDRs with only timeout set</dt><dd>Note that when a CDR only defines a lifetime. That is, no `multi use` and/or `expected amount`, the CDR will be used in input selection as soon as *ANY* funds reside on the given address. Developers are advised to use `multi use` as a safety measure, even when only one deposit is expected to arrive at an address.</dd></dl>

#### When should I set `multi use` to true?

When you expect multiple deposits to arrive at your address. This could be the case if, for example, you are displaying a CDR on your web page to receive donations.

Setting `multi use` to true also mitigates sweep attack scenarios, where an address would be used in input selection because it received funds prior to the actual expected deposit.

If you set `multi use` to true, an address is only used in input selection after its lifetime has ended. If an `expected amount` is defined and the address has that given amount, it will also be used in input selection, prior to the end of the lifetime.

#### When should I set an expected amount?

In the case where the value of the deposit is clear from the depositor's and receiver's point of view.

This would also be the case when you are for example withdrawing funds from an exchange. You can supply a CDR to the exchange which defines the amount to withdraw which equals the expected amount you are going to receive.

Situations where the expected amount is not known upfront are rare, therefore it is almost always possible to define it.

## Generating a new CDR

Use `AllocateDepositRequest()` to allocate a new CDR:
```go
// get current time
now, err := timesource.Time()
handleErr(err)
// define the time after which the CDR becomes invalid.
// (in this case after 2 hours)
now = now.Add(time.Duration(2) * time.Hour)
// allocate a new deposit address with timeout conditions.
conditions := &deposit.Request{TimeoutAt: &now, MultiUse: true}
cdr, err := acc.AllocateDepositRequest(conditions)
handleErr(err)

fmt.Println(cdr.AsMagnetLink())
// iota://MBREWACWIPRFJRDYYHAAME…AMOIDZCYKW/?t=1548337187&m=true&am=0
```

## Sending funds using a CDR
We want to determine whether, given a CDR, it makes sense to send a transactions or. Let's create a `SendOracle` for this purpose.  


### Determining whether to fulfill a CDR

We recommend developers to supply the `SendOracle` with multiple `OracleSource` objects for gaining a higher confidence in the `SendOracle`'s decision. An `OracleSource` is an object which tells the `SendOracle` whether it thinks a transaction should be sent or not. One could, for example, create an `OracleSource` which monitors the average confirmation time on  the network and from that computes a yes/no answer given a certain threshold.

In this example we only send funds if the current time is at least 30 minutes before the CDR's lifetime would end, i.e. we make the assumption, that our sent off bundle will confirm within a time frame of 30 minutes.

```go
threshold := time.Duration(30)*time.Minute
// timeDecider is an OracleSource
timeDecider := oracle_time.NewTimeDecider(timesource, threshold)
// we create a new SendOracle with the given OracleSources
sendOracle := oracle.New(timeDecider)
```

Assume we received the CDR as a magnet link into our application:

```go
cdr, err := deposit.ParseMagnetLink(cdrMagnetLink)
handleErr(err)

// we can now ask the SendOracle whether we should send the tx or not
ok, info, err := sendOracle.OkToSend(conds)
handleErr(err)
if !ok {
    logger.Error("won't send transaction:", info)
    return
}

// send the transaction:
// we assume that an expected amount was set in the CDR in this case
// and therefore the transfer is initialized with that amount.
bndl, err = acc.Send(cdr.AsTransfers())
handleErr(err)

fmt.Printf("sent off bundle to %s with tail tx %s\n", cdr.Address, bndl[0].Hash)
```