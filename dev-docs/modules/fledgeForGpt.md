---
layout: page_v2
page_type: module
title: Module - fledgeForGpt
description: how to use fledge with GPT
module_code : fledgeForGpt
display_name : Fledge for GPT
enable_download : true
sidebarType : 1
---

# Fledge (Protected Audience) for GPT Module

This module allows Prebid.js to support FLEDGE by integrating it with GPT's [experimental FLEDGE
support](https://github.com/google/ads-privacy/tree/master/proposals/fledge-multiple-seller-testing).

To learn more about FLEDGE in general, go [here](https://github.com/WICG/turtledove/blob/main/FLEDGE.md).

This document covers the steps necessary for publishers to enable FLEDGE on their inventory. It also describes
the changes Bid Adapters need to implement in order to support FLEDGE.

## Publisher Integration

Publishers wishing to enable FLEDGE support must do two things. First, they must compile Prebid.js with support for this module.
This is accomplished by adding the `fledgeForGpt` module to the list of modules they are already using:

```bash
gulp build --modules=fledgeForGpt,...
```

Second, they must enable FLEDGE in their Prebid.js configuration.
This is done through module level configuration, but to provide a high degree of flexiblity for testing, FLEDGE settings also exist at the bidder level and slot level.

### Module Configuration

This module exposes the following settings:

{: .table .table-bordered .table-striped }
|Name |Type |Description |Notes |
| ------------ | ------------ | ------------ |------------ |
|enabled | Boolean |Enable/disable the module |Defaults to `false` |
|bidders | Array[String] |Optional list of bidders |Defaults to all bidders |
|defaultForSlots | Number |Default value for `imp.ext.ae` in requests for specified bidders |Should be 1 |

As noted above, FLEDGE support is disabled by default. To enable it, set the `enabled` value to `true` for this module and configure `defaultForSlots` to be `1` (meaning _Client-side auction_).
using the `setConfig` method of Prebid.js. Optionally, a list of
bidders to apply these settings to may be provided:

```js
pbjs.que.push(function() {
  pbjs.setConfig({
    fledgeForGpt: {
      enabled: true,
      bidders: ['openx', 'rtbhouse'],
      defaultForSlots: 1
    }
  });
});
```

### Bidder Configuration

This module adds the following setting for bidders:

{: .table .table-bordered .table-striped }
|Name |Type |Description |Notes |
| ------------ | ------------ | ------------ |------------ |
| fledgeEnabled | Boolean | Enable/disable a bidder to participate in FLEDGE | Defaults to `false` |
|defaultForSlots | Number |Default value for `imp.ext.ae` in requests for specified bidders |Should be 1|

In addition to enabling FLEDGE at the module level, individual bidders must also be enabled. This allows publishers to
selectively test with one or more bidders as they desire. To enable one or more bidders, use the `setBidderConfig` method
of Prebid.js:

```js
pbjs.setBidderConfig({
    bidders: ["openx"],
    config: {
        fledgeEnabled: true,
        defaultForSlots: 1
    }
});
```

### AdUnit Configuration

All adunits can be opted-in to FLEDGE in the global config via the `defaultForSlots` parameter.
If needed, adunits can be configured individually by setting an attribute of the `ortb2Imp` object for that
adunit. This attribute will take precedence over `defaultForSlots` setting.

{: .table .table-bordered .table-striped }
|Name |Type |Description |Notes |
| ------------ | ------------ | ------------ |------------ |
| ortb2Imp.ext.ae | Integer | Auction Environment: 1 indicates FLEDGE eligible, 0 indicates it is not | Absence indicates this is not FLEDGE eligible |

The `ae` field stands for Auction Environment and was chosen to be consistent with the field that GAM passes to bidders
in their Open Bidding and Exchange Bidding APIs. More details on that can be found
[here](https://github.com/google/ads-privacy/tree/master/proposals/fledge-rtb#bid-request-changes-indicating-interest-group-auction-support)
In practice, this looks as follows:

```js
pbjs.addAdUnits({
    code: "my-adunit-div",
    // other config here
    ortb2Imp: {
        ext: {
            ae: 1
        }
    }
});
```

## Bid Adapter Integration

Chrome has enabled a two-tier auction in FLEDGE. This allows multiple sellers (frequently SSPs) to act on behalf of the publisher with
a single entity serving as the final decision maker. In their [current approach](https://github.com/google/ads-privacy/tree/master/proposals/fledge-multiple-seller-testing),
GPT has opted to run the final auction layer while allowing other SSPs/sellers to participate as
[Component Auctions](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#21-initiating-an-on-device-auction) which feed their
bids to the final layer. To learn more about Component Auctions, go [here](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#24-scoring-bids-in-component-auctions).

The FLEDGE auction, including Component Auctions, are configured via an `AuctionConfig` object that defines the parameters of the auction for a given
seller. This module enables FLEDGE support by allowing bid adaptors to return `AuctionConfig` objects in addition to bids. If a bid adaptor returns an
`AuctionConfig` object, Prebid.js will register it with the appropriate GPT ad slot so the bidder can participate as a Component Auction in the overall
FLEDGE auction for that slot. More details on the GPT API can be found [here](https://developers.google.com/publisher-tag/reference#googletag.config.componentauctionconfig).

Modifying a bid adapter to support FLEDGE is a straightforward process and consists of the following steps:

1. Detecting when a bid request is FLEDGE eligible
2. Responding with AuctionConfig

FLEDGE eligibility is made available to bid adapters through the `bidderRequest.fledgeEnabled` field.
The [`bidderRequest`](/dev-docs/bidder-adaptor.html#bidderrequest-parameters) object is passed to
the [`buildRequests`](/dev-docs/bidder-adaptor.html#building-the-request) method of an adapter. Bid adapters
who wish to participate should read this flag and pass it to their server. FLEDGE eligibility depends on a number of parameters:

1. Chrome enablement
2. Publisher participation in the [Origin Trial](https://developer.chrome.com/docs/privacy-sandbox/unified-origin-trial/#configure)
3. Publisher Prebid.js configuration (detailed above)

When a bid request is FLEDGE enabled, a bid adapter can return a tuple consisting of bids and AuctionConfig objects rather than just a list of bids:

```js
function interpretResponse(resp, req) {
    // Load the bids from the response - this is adapter specific
    const bids = parseBids(resp);

    // Load the auctionConfigs from the response - also adapter specific
    const auctionConfigs = parseAuctionConfigs(resp);

    if (auctionConfigs) {
        // Return a tuple of bids and auctionConfigs. It is possible that bids could be null.
        return {bids, fledgeAuctionConfigs};
    } else {
        return bids;
    }
}
```

An AuctionConfig must be associated with an adunit and auction, and this is accomplished using the value in the `bidId` field from the objects in the
`validBidRequests` array passed to the `buildRequests` function - see [here](/dev-docs/bidder-adaptor.html#ad-unit-params-in-the-validbidrequests-array)
for more details. This means that the AuctionConfig objects returned from `interpretResponse` must contain a `bidId` field whose value corresponds to
the request it should be associated with. This may raise the question: why isn't the AuctionConfig object returned as part of the bid? The
answer is that it's possible to participate in the FLEDGE auction without returning a contextual bid.

An example of this can be seen in the OpenX OpenRTB bid adapter [here](https://github.com/prebid/Prebid.js/blob/master/modules/openxOrtbBidAdapter.js#L327) or RTB House bid adapter [here](https://github.com/prebid/Prebid.js/blob/8fe7115021fd348d0f3b090da48c40c095078800/modules/rtbhouseBidAdapter.js#LL135C4-L135C4).

Other than the addition of the `bidId` field, the `AuctionConfig` object should adhere to the requirements set forth in FLEDGE. The details of creating an
`AuctionConfig` object are beyond the scope of this document.

## Related Reading

- [FLEDGE](https://github.com/WICG/turtledove/blob/main/FLEDGE.md)
- [Component Auctions](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#21-initiating-an-on-device-auction)
