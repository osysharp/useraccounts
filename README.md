# Osysharp.UserAccounts

Sign-in for a business app, done properly once. Every app that people sign in to needs the same things: sign in, sign
up (or not — some apps are invitation-only), a forgotten password, a changed password, a changed address, and proof
that an address is really theirs. Each of them is easy to get almost right. This kit gets them right, with the pages,
and the app writes **its roles and one setting**:

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Mail@0;
use Osysharp.UserAccounts@1;
use Osysharp.EmailTemplates@0;
use Osysharp.Ui;
use Osysharp.Workflow;
```

```osy
// model/auth.osy
using Osysharp.UserAccounts;

[Role] enum AppRole { Authenticator, Agent, Lead }       // Authenticator: the role the sign-in flow runs as
app.Accounts = new AccountsSetup { AppName = "Acme Support", SignupRole = AppRole.Agent, FirstAccountRole = AppRole.Lead };
```

That is all of it. The kit serves its own pages at its own routes — `/login`, `/sign-up`, `/forgot-password`,
`/reset-password/{token}`, `/verify-email/{token}`, `/cancel-address-change/{token}` and `/account`, the addresses its mail links to — and wires the app's
sign-in (`app.AuthBootstrap`, `app.Auth`) itself, so `osy user add` works and every flow is reachable by a signed-out
visitor.

To use a page of your own instead, declare a component with the kit page's name — it takes the page over, at the
kit's route or wherever your `[Route]` puts it:

```osy
[Route("/login")] [AllowAnonymous] [Render(CSR)]
component SignInPage() { render { MyFrame { SignInForm(); } } }
```

| Sign in | Sign up |
|---|---|
| ![The sign-in page](docs/sign-in.png) | ![The sign-up page](docs/sign-up.png) |

| Forgotten password | The signed-in account page |
|---|---|
| ![Asking for a reset link](docs/forgot-password.png) | ![One row per change: address, password, two-step sign-in, phone](docs/account.png) |

Every colour, face and size on these pages is a theme token, so the app's own theme restyles them.

## What it guarantees

- **The password hash is the sign-in flow's alone** — never read or written by anyone signed in, their own included,
  on `User` and on every type an app derives from it.
- **A wrong address takes as long as a wrong password**, and gets the same answer, so sign-in never tells a stranger
  which addresses have accounts.
- **A reset link** is a mailed secret, stored only as a hash, that works once and for one hour. Pressing the button
  twice mails once; using one link spends the others.
- **An address is proven by a message it received.** A change of address a person asks for takes effect only from a
  link sent to the NEW address, and the OLD address is told. Nothing else writes the address — not the account, not
  your staff, not a page or an API call (`deny update Email except via ConfirmAddressChange` on `User`).
- **The old address can call a move off.** Its mail carries a single-use "This wasn't me — cancel the change" link
  (`/cancel-address-change/{token}`): no password, because whoever asked has it. The new address's link and code then
  confirm nothing, and every browser signed in to the account is signed out. The account page's pending-address row
  has a Cancel of its own, for an address typed wrong.
- **Your desk can move an account for somebody who lost their address — on the record.** A person the app names
  (`policy ChangesAccountsFor => IsDesk;` — nobody until it does) calls `ChangeAddressFor(account, newAddress,
  reason)` — "ID checked at the desk". The account moves at once; who did it, when and why is kept; the old address is
  told it moved and the new one is welcomed — nothing to click in either.
- **Signing up first with somebody else's address gets you nothing.** The first time an account's address is proven,
  the link or code proves it only together with the password the account was made with (the sign-up and sign-in
  screens pass it along, so the person who made it is not asked again) — or with a NEW password, which hands the
  account to whoever holds the address and clears every way in set up before: the old password, two-step sign-in, a
  phone, linked providers.
- **Erasing an account leaves no way in.** `EraseAccount(account)`, called from your erasure's run, clears the address,
  the name, the password, two-step sign-in, a phone, linked providers, roles, sessions, every link and code sent to it and
  the outbox's copies of that mail — and keeps the row, which work it did still names.
- **One address, one account**, across every kind of account. A phone number may be proven by several accounts (one
  person, more than one account), each with its own code — and a number shared that way **signs in nobody by text**.
- **Sign-up gives only the roles the app chose** (`SignupRole`, and `FirstAccountRole` for the very first account),
  never one the visitor names. `SignupOpen = false` closes it.
- **Changing the password needs the current one**, and the account is told by mail.
- **The hash never reaches the audit trail.**
- **With two-step sign-in on, a password alone signs nobody in** — see below.
- **A passkey signs in with no password and no code**, and only on this app — see below.

## Two-step sign-in

After the password, a six-digit code from an authenticator app on the person's phone — or, when the phone is lost, one
of ten one-time recovery codes. Anybody may turn it on from the account page, and the app says who **must**:

```osy
public class AdminsUseCodes : SecondFactorPolicy {
  public override bool Required(User account) {
    return RoleGrant.Any(g => g.Member == account && (g.Role == AppRole.Admin || g.Role == AppRole.Lead));
  }
}
app.Accounts = new AccountsSetup { AppName = "Acme Support", SecondFactor = new AdminsUseCodes() };
```

Someone who must, and has not set it up, sets it up at sign-in — they are given no ticket until they have.

| The code, after the password | Setting it up | Recovery codes, shown once |
|---|---|---|
| ![The second step of signing in](docs/two-step-code.png) | ![A QR code and the key for an authenticator app](docs/two-step-setup.png) | ![Ten recovery codes on the account page](docs/recovery-codes.png) |

What holds:

- **No ticket before the second factor.** The password answers a short-lived, single-use challenge (kept only as a
  hash); the code turns it into a ticket. A mailed reset link sets a new password and signs nobody in, and the
  platform's own password sign-in (the CLI's) refuses such an account.
- **A code works once** — a code already used is refused, even inside its minute.
- **Guessing is bounded**: five wrong codes end a challenge, and ten on one account in fifteen minutes stop every code
  check for it, the right one included.
- **The authenticator secret is nobody's but the sign-in flow's**, the account's own included, and is stored sealed
  with the app's own key.
- **Recovery codes** are kept only as keyed fingerprints, work once, and a new set spends the old one. Using one — and
  turning two-step sign-in on or off — mails the owner.
- **Turning it off, or new recovery codes, take the password and a current code**; so, while it is on, do a new
  password or address.

## Settings

```osy
app.Accounts = new AccountsSetup {
  AppName = "…",              SignupOpen = true,           SignupRole = AppRole.…,     FirstAccountRole = AppRole.…,
  MinPasswordLength = 10,     RequireVerifiedEmail = false, HomePath = "/",
  Mails = new AccountMails(), // derive to write your own words for each mail
  Kinds = new AccountKinds(), // derive to make your own kind of account on sign-up
  SecondFactor = new SecondFactorPolicy(),   // derive to say who must use two-step sign-in
  AddressProof = new AddressProofPolicy(),   // derive to say whose address must be proven before a first sign-in
  LogoUrl = "",               // the app's mark on every sign-in screen: File.Url("public/brand/logo.svg"), or https
  Marks = new ProviderMarks(), // derive to give a sign-in provider's buttons its own mark (GitHub, Google, Microsoft are built in)
};
```

Every "Continue with …" and "Link …" button shows its provider's mark. GitHub, Google and Microsoft have theirs built in;
any other provider shows the plain sign-in glyph until the app gives it one, from its own icons:

```osy
public class DeskMarks : ProviderMarks {
  public override Icons For(string provider) { return provider == "Acme SSO" ? Icons.Acme : base.For(provider); }
}
```

## Staff and their roles

A new account gets the role sign-up gives it. A manager giving staff their roles and taking them back — with a record of
who changed what, deactivating somebody who left, signing a person out everywhere — is **Osysharp.UserManagement**, the user
manager. An app with more than one role usually adds it: `use Osysharp.UserManagement@0;`.

## An account for somebody who never signs in

A child a parent books for, a walk-in at the desk: `MakeManagedAccount("Sam")` makes a real account with no password and
no address anybody can mail, recorded with who made it and who looks after it. Who may is the app's to say:
`policy MakesManagedAccounts => IsMember;`. Later, `GiveManagedAccountAnAddress(account, "sam@example.com")` sends a
link; the account moves there once it is opened, and Sam chooses a password like anybody else.

## Your own kind of account

What the app knows about its people goes on a type derived from `User`. Tell sign-up to make it:

```osy
entity Employee : User {
  [MaxLength(60)] string Team = "";
  security {                                     // its own grants — inside the kit's rules, which still hold
    allow read when IsAuthenticated || IsAuthenticator;
    allow create when IsAuthenticator;
    allow update when IsAuthenticated || IsAuthenticator where Id == user.Id;
    allow update when IsAuthenticator;
  }
}
public class StaffKinds : AccountKinds {
  public override User New(string email, string name) { return new Employee { Email = email }; }
}
app.Accounts = new AccountsSetup { AppName = "Acme Support", Kinds = new StaffKinds() };
```

In security rules `user` is the base `Principal`; `user is Employee` tells the kinds apart.

## Mail

Reset links, verifications and notices go through [Osysharp.Mail](https://osyrin.com/templates/kits/mail/)'s outbox. Until the app names a
sender, mail is **held** — kept, with the reason — and the app still signs people up and resets passwords. Name one,
for example Resend:

```osy
app.Mail = new MailSetup {
  Sender = new ResendSender { ApiKey = Security.IsSecretSet("ResendApiKey") ? Secret.ResendApiKey : "" },
  From = "Acme <noreply@acme.com>",
};
app.Secrets = [ new Secret("ResendApiKey") ];
```

## Passkeys

A passkey is a key pair the person's phone, laptop or security key makes for this app and unlocks with a PIN, a
fingerprint or a face. It signs in with no address typed, no password and no code — it is both steps at once — and a
look-alike page cannot collect it, because the browser offers it only to the app it was made for.

Every app on the kit has it: "Sign in with a passkey" on the sign-in page (where the browser can make passkeys), and a
**Passkeys** section on the account page to add several — the Mac, the phone, a hardware key in a drawer as the spare —
name them, and remove them. An account the app requires two-step sign-in of may set a passkey up instead of an
authenticator app. Once an account has a passkey it may **turn password sign-in off**: the password is removed, so
nothing that knows only passwords (this app's sign-in, the command line's) can sign it in. A lost device is recovered by
another passkey, or by "Forgot your password?", whose mailed link sets a password again.

What holds:

- **Every answer is checked against a challenge the sign-in flow minted** — kept as a hash, alive five minutes, spent by
  the first answer that comes back, right or wrong. An add's challenge is never a sign-in's.
- **The browser's answer must be for this app** — its origin and its host — **and the device must have checked the
  person** (user verification is required, not preferred).
- **A signature counter that does not move forward is refused** — the key may have been copied. (Synced passkeys report
  zero every time, which is not a regression.)
- **A passkey is one account's**: its id is unique here, and its user handle must be its account's.
- **The last way in is never removed**: a passkey goes only while a password, another passkey or a linked provider
  remains, and the password only while a passkey does.
- **Adding or removing one takes proof** — the password (and a code, with two-step sign-in on), or one of the account's
  passkeys — and mails the owner. A stranger who signed up first with somebody's address loses their passkey with
  everything else the moment the owner proves the address.

The checks a browser's answer needs — CBOR, COSE keys, ES256 / RS256 / EdDSA signatures — are the platform's
(`WebAuthn.*`); the prompt is the browser's (`Passkey.Create`, `Passkey.Get`). Everything that decides about an account
is this kit's `model/Passkeys.osy`, in Osy#.

## Building your own on top of it

The pieces are separate on purpose. The forms (`SignInForm` — which carries the code step and the set-up step —
`SignUpForm`, `ForgotPasswordForm`, `ResetPasswordForm(token)`, `VerifyEmailResult(token)`, `AccountSettings`,
`SecondFactorEnrolment`, `RecoveryCodeList`) go in any frame of your own. The flows are ordinary functions — `SignIn`,
`SignUp` (both answer a `SignInStep`: a ticket, a challenge, or "set it up first"), `SignInWithCode`, `RequestPasswordReset`,
`ResetPassword`, `ChangePassword`, `ChangeEmail`, `CancelAddressChange`, `CancelMyAddressChange`, `VerifyEmail`, `EnableSecondFactor`, `DisableSecondFactor`,
`NewRecoveryCodes`, `StartPasskeySignIn`, `SignInWithPasskey`, `AddPasskey`, `RemovePasskey`, `RemovePassword` — and an app that states its own `app.AuthBootstrap` replaces the kit's wiring whole. A sign-in that is
not a password (a company's identity provider, a code by text message) builds on
[Osysharp.Identity](https://osyrin.com/templates/kits/identity/) directly.

**A door of your own** — say an application to a waitlist, with `SignupOpen = false` — is built from the kit's own
pieces: `SignUpAtDoor` makes the account (answering a taken address exactly like a new one), `TextPhoneCode` and
`ProvePhoneCode` prove a phone for the account your door has established, and `AccountsSetup.OwnDoor` sends a provider
sign-in for an account your door has not finished with back to the door instead of signing it in.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;

[Test(AssistedDesk)]
void the_desk_moves_the_account_at_once_and_records_who_and_why() {
  runas (Lead) { ChangeAddressFor(User.Single(u => u.Email == "bo@desk.test"), "Bo.Home@desk.test", "ID checked at the desk"); }
  runas (AuthBootstrap) {
    // Moved now — no confirmation from either address.
    Assert.Equal("bo.home@desk.test", User.Single(u => u.DisplayName == "Bo").Email);
    Assert.False(User.Any(u => u.Email == "bo@desk.test"));
    // The record: from where, to where, why, and by whom.
    var change = AddressChangeRequest.Single();
    Assert.Equal("bo@desk.test", change.OldAddress);
    Assert.Equal("bo.home@desk.test", change.NewAddress);
    Assert.Equal("ID checked at the desk", change.Reason);
    Assert.Equal(User.Single(u => u.Email == "lead@desk.test").Id, change.MadeBy);
    Assert.Equal(AddressChangeState.Moved, change.State);
    // Both addresses told; nothing to click in either.
    Assert.Equal(1, MailsTo("bo@desk.test", "Your Kit Test Desk address was changed"));
    Assert.Equal(1, MailsTo("bo.home@desk.test", "Your Kit Test Desk account now uses this address"));
    Assert.Equal("", TokenIn("bo.home@desk.test", "/verify-email/"));
    Assert.Equal("", TokenIn("bo@desk.test", "/verify-email/"));
  }
  // Bo signs in with the new address and the password he had.
  runas (AuthBootstrap) { Assert.NotEqual("", SignIn("bo.home@desk.test", "correct-horse-2").Ticket); }
}
```

From [`tests/assisted-change.test.osy`](tests/assisted-change.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;

[Test(People)]
void opening_the_link_proves_the_address() {
  runas (AuthBootstrap) { SignIn("ada@desk.test", "correct-horse-1"); }   // an unproven account is sent a link
  string token = "";
  runas (AuthBootstrap) { token = TokenIn("ada@desk.test", "/verify-email/"); }
  Assert.NotEqual("", token);
  // Ada's account had its password before its address was proven, so the link proves it WITH that password (her first
  // proof — tests/pre-hijack.test.osy); proven, and signed in as a sign-in would be.
  SignInStep proven = new SignInStep();
  runas (AuthBootstrap) { proven = VerifyEmail(token, "correct-horse-1"); }
  Assert.NotEqual("", proven.Ticket);
  runas (AuthBootstrap) { Assert.NotNull(User.Single(u => u.Email == "ada@desk.test").EmailVerifiedAt); }
  // Once only: opened again it signs nobody in, and says the address is already the account's.
  SignInStep again = new SignInStep();
  runas (AuthBootstrap) { again = VerifyEmail(token, "correct-horse-1"); }
  Assert.True(again.AlreadyProven);
  Assert.Equal("", again.Ticket);
}
```

From [`tests/email.test.osy`](tests/email.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.UserAccounts;

[Test(Leaving)]
void an_erased_account_cannot_sign_in_reset_or_be_reached() {
  // A reset link mailed before the erasure — it must not work after it.
  runas (AuthBootstrap) { RequestPasswordReset("ada@desk.test"); }
  string token = "";
  runas (AuthBootstrap) { token = TokenIn("ada@desk.test", "/reset-password/"); }
  Assert.NotEqual("", token);
  runas (Ada) { new AccountErasure { Account = User.Single(u => u.Email == "ada@desk.test") }; }
  runas (AuthBootstrap) {
    Assert.False(User.Any(u => u.Email == "ada@desk.test"));
    var erased = User.Single(u => u.DisplayName == "" && u.Email.EndsWith("@erased.invalid"));
    Assert.False(erased.IsActive);
    Assert.Null(erased.PasswordHash);
    Assert.False(RoleGrant.Any(g => g.Member == erased));
    // …and nothing mailed to her is left in the outbox.
    Assert.Equal(0, MailsTo("ada@desk.test", ""));
  }
  runas (AuthBootstrap) { Assert.Equal("", SignIn("ada@desk.test", "correct-horse-1").Ticket); }
  runas (AuthBootstrap) { Assert.Throws(() => ResetPassword(token, "a-brand-new-one")); }
}
```

From [`tests/erasure.test.osy`](tests/erasure.test.osy).

## Learn more

- [The Osysharp.UserAccounts reference](https://osysharp.com/reference/security/accounts-kit/)
- [Osysharp.UserAccounts in the package catalogue](https://osysharp.com/packages/#osysharp-useraccounts)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
