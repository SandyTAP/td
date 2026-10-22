# Layer 232 schema review

Source: official Telegram Desktop `dev@850fb90eb6db96c124b6da1349060f9ed00f4fe4`.
The last change to `Telegram/SourceFiles/mtproto/scheme/api.tl` is the same
commit; manifest.json locks its Git blob
`8319b7df634d145390c299404129fd161a5dbd2a` and normalized SHA-256
`76922ce265441d3f2f5188f56b69b41e1cb11354657ea547707507f8c3b93826`.
The TDesktop source path is clean and this schema commit is reachable from
`origin/dev`. No client patch is part of the schema import.

Relative to Layer 231 the schema adds 24 constructors and changes seven
constructor IDs. Canonical moves to 232; exact profiles 225–231 remain.

## Constructor changes

| Constructor | Layer 231 | Layer 232 |
|---|---|---|
| `channel` | `#d49f34c6` | `#7ad15240` (adds `linked_community_id`) |
| `channelFull` | `#a04e8d3a` | `#c4c4dd40` (adds `share_theme`) |
| `userFull` | `#6cbe645` | `#b90c2afd` (adds `wallet_card`, `note`) |
| `messageReplyHeader` | `#1b97dd66` | `#ddb5ab75` (adds `quote_rich`) |
| `inputReplyToMessage` | `#3bd4b7c2` | `#75c1ad70` (adds `quote_rich`) |
| `wallet.nftItem` | `#3441fd1c` | `#173dfaf3` (adds `gift`) |
| `payments.getStarGiftWithdrawalUrl` | `#d06e93a8` | `#c8ca817b` (`password:InputCheckPasswordSRP` → `tmp_password:bytes`) |

New constructors: `messageActionNoPaidMessagesTokenUsed`,
`walletTransactionPeerChannel`, `walletTransactionPeerTelegramGift`,
`exportedNoPaidMessagesToken`, `shareThemeDefault`, `shareThemeEmojiBackdrop`,
`shareThemeGift`, `inputShareThemeGift`, `shareThemeBackdropOption`,
`shareThemeBackdropsNotModified`, `shareThemeBackdrops`,
`inputWalletCardSticker`, `wallet.cardSticker`, `wallet.card`,
`payments.starGiftDepositPayload`, `contacts.exportNoPaidMessagesToken`,
`contacts.importNoPaidMessagesToken`, `messages.setShareTheme`,
`messages.getShareThemeBackdrops`, `payments.withdrawStarGift`,
`payments.depositStarGift`, `payments.displayStarGift`,
`payments.deleteStarGift`, `wallet.setCardStickers`.

No update constructor or `pts`/`pts_count` field changed; the MTProto
service schema is unchanged.

## Compatibility decisions

| Scenario | Decision |
|---|---|
| `payments.getStarGiftWithdrawalUrl` in exact 232 | The request carries `tmp_password:bytes`. A non-empty temporary password is a Telegram temporary-password proof this fork does not issue, so a server without `account.getTmpPassword` must refuse it rather than treat it as a 2FA proof. |
| Historical `password:InputCheckPasswordSRP` projected to canonical 232 | Discard. The SRP material has no canonical counterpart. Dropping it keeps the empty-password check authoritative: an account that owns a 2FA password still fails closed, and a passwordless account is unaffected. The password is never silently promoted into a temporary password. |
| Canonical `tmp_password` projected to exact 225–231 | Reject. The temporary password cannot be encoded as the legacy SRP challenge, and inventing an empty SRP challenge would downgrade an authenticated request to an unauthenticated one. |
| Historical profile without `tmp_password` projected to canonical 232 | Default to empty bytes. Empty means "no temporary password supplied" and keeps the empty-password check semantics. |
| New constructors in historical profiles | Unavailable. No alias, no byte bridge. A historical client cannot call `payments.deleteStarGift`, `messages.setShareTheme`, `wallet.setCardStickers` and the other new methods. |
| New fields on existing constructors | Additive. Historical profiles keep their own constructor and field set; canonical carries the new fields. |
| Reshuffled constructor IDs | Every exact profile keeps its own wire ID. `channel`, `channelFull`, `userFull`, `messageReplyHeader`, `inputReplyToMessage` and `wallet.nftItem` are retagged per profile, never rewritten in place. |

Only the `payments.getStarGiftWithdrawalUrl` request projections override the
generic defaults. The remaining new obligations are mechanical reuse: a new
constructor in a historical profile is unavailable, and a new field in a
historical profile is dropped or rejected exactly as the reviewed Layer 231
policy already requires.

## Verification matrix

Check literal old/new method and result constructor IDs for the seven
reshuffled constructors; exact 225–232 request/result round trips for
`payments.getStarGiftWithdrawalUrl` with and without a temporary password;
reject canonical→historical temporary passwords; default historical requests
to an empty temporary password; keep every other exact profile wire ID
unchanged; retain the 228/229 PageBlock storage readers. Run deterministic
generation, protocol tests and `go vet`, then downstream RPC/edge tests and
command builds against the published module without a permanent local
`replace`. This protocol change allocates no PTS/QTS, durable outbox event, or
session state.

## 2026-10-22 verification

On Linux/amd64, Go 1.27.1:

- Schema import verified against the locked Git blob and normalized SHA-256;
  `canonical_layer` moved to 232 and `layer-232.tl` was written with its
  provenance.
- Policy audit: 249 retained entries, zero stale, 21 new obligations, all of
  them the `payments.getStarGiftWithdrawalUrl` projections reviewed above. The
  reviewed policy document now holds 270 entries.
- Generation is deterministic and reproduced the canonical `tg` model plus the
  `tlprofile` sidecar with `Profile232` and `ProfileCanonical = 232`.
- The locked generator budgets were re-measured for the eighth profile:
  714 projected families, 2538 rewrite variants, 957 unavailable variants,
  259 body obligations, 4 result obligations, 1899/1148/969 execution plans,
  1527 typed codec wires and 399 unprofiled wrapper invariants.
