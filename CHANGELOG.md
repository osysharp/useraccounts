# Osysharp.UserAccounts — changes

## 2.0.1
- A presence refused for want of a factor says "Add a passkey or two-step sign-in to approve this." — the kit's own words
  for any act an app asks presence for. 2.0.0 said "…to approve deploys", one app's act, to every app.
- The presence refusals (`StartPresence`, `ProvePresence`, an act presence is not proven for) and "Two-step sign-in is
  required here — sign in again." are translated into Swedish and Arabic in the kit, so an app that speaks either has
  nothing of the kit's left to translate.

## 2.0.0
The code step of a sign-in answers a `SignInStep` in place of a ticket string, so it can say "set a factor up next". The
calls that changed (the old spelling is gone; nothing forwards to it):
- `SignInWithCode(string challenge, string code) -> string` → `SignInWithCode(string challenge, string code) -> SignInStep`.
- `SignInWithTextedCode(SignInChallenge pending, string code) -> string` → `SignInWithTextedCode(SignInChallenge pending,
  string code) -> SignInStep`.

What a caller does with the step it gets back:
- `Ticket` set: signed in, as the string was — hand it to `Session.SignIn` (`SignInWithCode(challenge, code).Ticket`).
- `MustEnrol`: the code was right and this account must set up a second factor that counts before it is signed in (it chose
  a texted code under a rule that does not count one). Set one up with `EnableSecondFactor` or `AddPasskey`, passing the step's
  `Enrolment` token as the proof of the first step; `Account` is the address to label the authenticator app with. No ticket.
- Every field empty: the code does not fit — ask for it again on the same challenge (the empty string, before).
- A thrown `ValidationException`, as before: the challenge is spent or expired, the account has had too many wrong codes, or
  (a texted code) the account holds a factor of its own and must use it — show its sentence and start again.

`use Osysharp.UserAccounts@2;` in place of `@1`; nothing else in an app changes unless it calls `SignInWithCode` itself.

`RecordMiss` is `internal`: only the kit's own code checks write a miss. It had no caller outside the kit.

Also in 2.0.0, presence for one act:
- Presence: proof, once and for one named act, that the signed-in person is acting, kept as a `PresenceProof`.
  `StartPresence(act)` starts a passkey ceremony for this account and act (only its own passkeys, user verification
  required); `ProvePresence(act, …)` takes the answer, or the password and a current authenticator code, and answers a
  proof; `TakePresence(proof, act)` is true once, for the same account and act, within five minutes. A recovery code, a
  texted or mailed code, a provider and a ticket's claims never prove presence; a code already used (at sign-in or for
  presence) never proves again, and every wrong password, code or passkey counts toward the account's lockout.
  `StepCountsAsPresence(way)` says which second steps count, from the same list the two-step rule asks.
- `EndPresenceOf(account)` ends an account's live proofs; a sign-out everywhere, a new password and a reset second step
  call it.
- A presence ceremony is never a sign-in or a confirmation, and those are never presence.

Also in 2.0.0 (prepared as 1.17.0, which was never published):
- Two-step sign-in for everyone (`AccountsSetup.SecondFactorForAll`, an organisation's lock): every account signs in with an
  authenticator code, a passkey or, where `ProviderFactorCounts` allows it, the provider's own second factor through the
  organisation's pin (its own sign-in client, or a Microsoft tenant its provider-only rule names; never Google). A texted code never counts. An account with
  none sets one up at sign-in.
- `AccountsSetup.SignInHours` (1–12, an organisation's range): every ticket ends that many hours after the sign-in. 12, the
  default, adds no end.
- `SignInRulePolicy.SecondFactorForAll(account)` and `SignInHours(account)`: an app's override adds two-step sign-in and
  lowers the hours, never lifts the app's own.
- Every ticket the kit issues comes from one function; a changed password's fresh ticket needs the second step too (an
  account that must use it and gave no code is refused, and nothing is changed).
- While a sign-in has an end of its own (hours under 12), it starts no MCP or CLI sign-in until the platform admits the
  kit's capped tickets there.
- An account that has a second factor of its own (an authenticator app or a passkey) uses it: a sign-in that needs a second
  step never sends it to set one up, an enrolment token is refused for it, and `EnableSecondFactor` and `AddPasskey` add a
  factor to it only past one of its own, never on the password alone. `EnableSecondFactor` takes `passkeyToken` and
  `passkeyResponse` for an account that signs in with a passkey. `RemovePasskey` too takes one of the account's passkeys, or
  the password and the authenticator's code, never the password alone.
- `SignInWithCode` answers a `SignInStep` (its `Ticket`, as the code step's answer was) in place of a string. An account
  that must use two-step sign-in and chose a texted code is texted first, and past the code `SignInWithCode` answers
  `MustEnrol` (the set-up), never a ticket; its password alone no longer reaches the set-up. The sign-in page goes on to it.
- A texted second step is a factor the password alone does not get past: `EnableSecondFactor` and `AddPasskey` refuse an
  account that holds one on its password (it turns the texted step off first, which asks for a texted code; under a rule
  that asks for a factor, its sign-in texts it and then sets one up). `SetSmsSecondStep(…, true)` refuses an account that holds a
  passkey, and `ProvePhone` and `ProvePhoneCode` no longer move a number that is the account's texted step.
- The platform holds every ticket of the app to `SecondFactorForAll` and `SignInHours`, its own password and provider doors
  too, whatever `app.AuthBootstrap` the app states.
  Needs platform 0.24.0 (`Security.SignInStepRequired`, the four-argument `Security.IssueJwt`): the release these ship in.

## 1.13.0
- Its organization-policy settings name their area from the platform's list (`Area = OrgPolicyArea.SignIn`), not as text.
  Needs platform 0.23.0 (`minPlatform "0.23.0"`): an older platform refuses this version by its floor.
