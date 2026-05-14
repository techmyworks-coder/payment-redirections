# Payment Redirection — Before / After

Recipient data (`vpa`, `pn`, `am`) is unchanged in both files. Only the deeplink wrapping changed — different schemes, cascade fallbacks, platform-specific URLs.

Defaults used in the examples below:
- `vpa` = `9145899828@okbizaxis`
- `pn` = `Man Mohan Light Decoration`
- `am` = `1`
- `tn` = `Payment`

---

## pp.html — PhonePe

### Before

Single URL, fired identically on Android and iOS, no fallbacks.

```
phonepe://native?id=p2pContactChat&data=<base64>
```

The `<base64>` decodes to:
```json
{
  "contactInfo": {
    "type": "VPA",
    "name": "Man Mohan Light Decoration",
    "vpa": "9145899828@okbizaxis",
    "isOnPhonePe": true,
    "isUpiEnabled": true,
    "hasExternalVpa": true
  },
  "withSheetExpanded": true,
  "shouldShowUnsavedBanner": false,
  "shouldAutoShowKeyboard": true,
  "initialTab": "SEND",
  "sendParams": { "initialAmount": 100, "note": "Payment" },
  "validateDestination": false
}
```

### After

Platform-specific cascade (1.5s between each URL).

**Android** (2 URLs, identical payload):
```
1. intent://native?id=p2pContactChat&data=<base64>#Intent;scheme=phonepe;package=com.phonepe.app;end
2. phonepe://native?id=p2pContactChat&data=<base64>
```

**iOS** (2 URLs):
```
1. phonepe://pay?pa=9145899828%40okbizaxis&pn=Man%20Mohan%20Light%20Decoration&am=1&tn=Payment&cu=INR
2. https://phon.pe/pay?pa=9145899828%40okbizaxis&pn=Man%20Mohan%20Light%20Decoration&am=1&tn=Payment&cu=INR
```

### What changed

| | Before | After |
|---|---|---|
| iOS behavior | Fired Android-only `phonepe://native` (likely broken) | Uses `phonepe://pay` (works on iOS) |
| Android primary | `phonepe://` scheme | `intent://` wrapper (more reliable from browser) |
| Android fallback | none | falls back to `phonepe://` after 1.5s |
| iOS fallback | none | Universal Link `https://phon.pe/pay?…` after 1.5s |
| Payload | unchanged | unchanged |

---

## ptm.html — Paytm

### Before

Single URL, fired identically on Android and iOS, no fallbacks. Targeted Paytm's chat-with-VPA screen.

```
paytmmp://chat?featuretype=start_chat&userType=VPA&vpa=9145899828@okbizaxis&txnCategory=VPA2VPA&amount=1&source=deeplink
```

### After

Platform-specific cascade (1.5s between each URL). Targets Paytm's send-money / UPI search screen, not chat.

**Android** (3 URLs):
```
1. intent://cash_wallet?featuretype=money_transfer&pa=9145899828%40okbizaxis&pn=Man%20Mohan%20Light%20Decoration&am=1&tr=Payment&tn=Payment&cu=INR#Intent;scheme=paytmmp;component=net.one97.paytm/net.one97.paytm.landingpage.activity.DeepLinkActivity;end

2. paytmmp://cash_wallet?featuretype=money_transfer&pa=9145899828%40okbizaxis&pn=Man%20Mohan%20Light%20Decoration&am=1&tr=Payment&tn=Payment&cu=INR

3. upi://pay?pa=9145899828%40okbizaxis&pn=Man%20Mohan%20Light%20Decoration&am=1&tn=Payment&cu=INR
```

**iOS** (2 URLs):
```
1. paytmmp://pay?pa=9145899828%40okbizaxis&pn=Man%20Mohan%20Light%20Decoration&am=1&tr=Payment&tn=Payment&cu=INR

2. upi://pay?pa=9145899828%40okbizaxis&pn=Man%20Mohan%20Light%20Decoration&am=1&tn=Payment&cu=INR
```

### What changed

| | Before | After |
|---|---|---|
| Flow type | Chat screen (`paytmmp://chat`) | Send-money screen (`cash_wallet?featuretype=money_transfer`) |
| Where user lands | Paytm chat thread with VPA | Paytm UPI search/confirm screen, VPA + amount pre-filled |
| iOS behavior | Likely broken (Android-only chat scheme) | Works — `paytmmp://pay` is iOS-registered |
| Android fallbacks | none | 3-step cascade ending in generic `upi://pay` |
| iOS fallbacks | none | falls back to generic `upi://pay` |
| Compatibility with merchant VPAs | Often refused (chat is P2P-only) | Works with any VPA, including merchant handles |

---

## Common cascade behavior

Both files now use the same redirect engine:

```
fire URL[0]
  ↓ wait 1.5s
fire URL[1]
  ↓ wait 1.5s
fire URL[2]   (Paytm Android only)
  ↓ wait
show "didn't open?" button (Android: 4–5.5s after start; iOS: shown immediately)
```

If the first URL successfully launches the app, the page is backgrounded and subsequent URLs do nothing. The cascade exists so that a missing scheme registration doesn't dead-end the user.

---

## Why the change

The VPA in your test data (`9145899828@okbizaxis`) is a merchant handle. PhonePe and Paytm both treat merchant VPAs differently from personal VPAs:

- **Chat-flow deeplinks** (`paytmmp://chat`, `phonepe://native?id=p2pContactChat`) are designed for P2P (person-to-person). NPCI's resolver flags merchant VPAs and refuses to open a chat thread for them.
- **Send-money deeplinks** (`paytmmp://cash_wallet?featuretype=money_transfer`, `phonepe://pay`) work for any VPA — personal or merchant — because they target the standard UPI confirm screen, which has no P2P constraint.

The "after" URLs land users on the same screen they'd see if they pasted the VPA into Paytm/PhonePe manually. The "before" URLs targeted a chat experience that only fires for personal VPAs.
