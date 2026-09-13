# 03 — Acceptance criteria

Build against these. Each is a behaviour you can assert on; each should map to at least
one automated test that fails before the code satisfying it exists.

The failure mode of this system is **silent disclosure** — nothing crashes, nothing logs
an error, a credential is simply readable when it should not be. Criteria in
[§2](#2-denial-must-be-uniform) and [§3](#3-isolation-between-owners) are the ones that
catch it, and they are the ones an implementation is most likely to satisfy by accident
in the happy path and fail under a case nobody thought about.

Where a criterion says *identical*, an implementation MUST NOT weaken the test to make a
change pass. If a test asserting identical responses starts failing, the response
stopped being identical — that is the test working.

---

## 1. The state machine

- [ ] **1.1** A newly created secret is closed. Reading it is denied.
- [ ] **1.2** After approval, reading returns exactly the stored value.
- [ ] **1.3** Each successful read decrements the remaining budget by one.
- [ ] **1.4** When the budget reaches zero, the next read is denied, even though the
      time window is still open.
- [ ] **1.5** Once the time window has passed, reads are denied, even with budget
      remaining.
- [ ] **1.6** Revoking an armed secret denies the very next read.
- [ ] **1.7** Rotating the value of an armed secret closes it, and the previous value is
      never served again.
- [ ] **1.8** After rotating and re-approving, reads return the new value.
- [ ] **1.9** Approving an already-armed secret resets the budget rather than adding to
      it. *Approve with 5, read once, approve with 5 again → 5 remaining, not 9.*
- [ ] **1.10** A successful read increments the lifetime read count and updates the
      last-read time.
- [ ] **1.11** With a budget of one and several concurrent readers, exactly one receives
      the value and all others are denied. *Assert on the counts, not on which one won.*
- [ ] **1.12** No secret is ever removed, closed or altered by the passage of time alone
      other than its window closing. *Leave a secret unread for longer than any
      configured interval and confirm it still exists.*

## 2. Denial must be uniform

- [ ] **2.1** Reading an unknown name, a closed secret, and a budget-exhausted secret
      produce responses identical in status, body and headers.
- [ ] **2.2** Reading a secret owned by a different user is identical to all of the
      above.
- [ ] **2.3** Reading a secret whose stored value cannot be decrypted is identical to
      all of the above — not a server error. *Point a second instance with a different
      key at the same store and confirm.*
- [ ] **2.4** No denial carries a response body.
- [ ] **2.5** No denial carries an authentication-challenge header.
- [ ] **2.6** A failed decryption does not consume budget.
- [ ] **2.7** After a threshold of denials, further attempts receive a distinct
      rate-limit response.
- [ ] **2.8** The threshold is global. *Send each denial from a different forwarded
      client address; the limit still trips. A per-client limiter passes every other
      criterion here and is free to evade.*
- [ ] **2.9** Denials for different names share the one counter. *Enumeration walks
      names, not addresses.*
- [ ] **2.10** A tripped limit still serves an armed token. *Otherwise an
      unauthenticated caller can halt every script by asking for names that do not
      exist.*
- [ ] **2.11** Successful reads never count toward the limit. *Approve a budget larger
      than the denial threshold and read it out completely.*

## 3. Isolation between owners

- [ ] **3.1** Listing shows only the requesting owner's secrets.
- [ ] **3.2** Approve, revoke, rotate and delete each fail for a secret owned by someone
      else.
- [ ] **3.3** For every one of those operations, the response for *another owner's
      secret* is identical to the response for *a name that does not exist*.
- [ ] **3.4** A failed cross-owner operation leaves the target completely unchanged —
      still armed if it was armed, same value, same budget.
- [ ] **3.5** No page or response reveals another owner's secret names.

## 4. Identity

- [ ] **4.1** Every operation other than reading and the health check is refused without
      an established identity.
- [ ] **4.2** An absent or empty identity is refused rather than resolving to a default
      or first user.
- [ ] **4.3** Reading works with no identity whatsoever. *This is the one route scripts
      use; if it requires a session, nothing works.*
- [ ] **4.4** Any development identity bypass refuses to start when configured for a
      non-local public URL.

## 5. Confidentiality

- [ ] **5.1** The plaintext value does not appear anywhere in the raw storage. *Search
      the database file's bytes for it.*
- [ ] **5.2** A modified ciphertext raises an error rather than returning garbage.
- [ ] **5.3** No secret value appears in any log line, at any log level, through
      creation, approval, reading and rotation.
- [ ] **5.4** No secret value appears in any rendered page, including the owner's own
      dashboard.
- [ ] **5.5** No audit record contains a secret value, in any field.
- [ ] **5.6** The service refuses to start without a valid encryption key.

## 6. Audit

- [ ] **6.1** A successful read writes a record with the outcome, source address and
      remaining budget.
- [ ] **6.2** A denied read writes a record, including for names that do not exist.
- [ ] **6.3** Create, approve, revoke, rotate and delete each write a record.

## 7. The read response

- [ ] **7.1** The body is exactly the value — no envelope, no quotes, no added
      whitespace or newline. *Assert byte equality, not "contains".*
- [ ] **7.2** The content type is plain text.
- [ ] **7.3** No-store cache, no-referrer referrer policy and nosniff options are all
      present.

## 8. Input handling

- [ ] **8.1** Names outside the permitted pattern are refused with an explanation, and
      nothing is stored. *Cover: empty, leading hyphen, trailing hyphen, uppercase,
      underscore, space, dot, over-length.*
- [ ] **8.2** Creating a name that already exists is refused and does not overwrite.
- [ ] **8.3** A non-positive or non-numeric approval window or budget is refused with an
      explanation, and the secret stays closed.
- [ ] **8.4** Rejections are shown to the user, not merely implied by nothing having
      happened. *Assert on the message reaching the page, not just on the resulting
      state.*

## 9. Operational

- [ ] **9.1** The service refuses to start on invalid configuration, rather than failing
      later at the point of use.
- [ ] **9.2** The health check reveals nothing about stored content.
- [ ] **9.3** Restarting the service preserves all secrets and their armed state.
- [ ] **9.4** Absolute URLs are built from configuration, not from the incoming request.
      *Send a request with a misleading host header and confirm generated URLs ignore
      it.*
- [ ] **9.5** Secrets survive an upgrade of the service. *Write with one version, read
      with the next.*

---

## Coverage note

An implementation passing §1 and §7 works. An implementation passing §2, §3 and §5 is
*safe*. The second set is where the value is, and where a reimplementation is most
likely to fall short — not through carelessness, but because a natural implementation of
"return 404 if not found, 401 if not armed" reads as obviously correct and is an
existence oracle.
