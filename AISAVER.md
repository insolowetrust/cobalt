# AI Saver Cobalt fork

This is a fork of [imputnet/cobalt](https://github.com/imputnet/cobalt) for AI Saver. It carries an Instagram fix used until upstream ships an equivalent change. Nothing in this repository should be deployed until Maksym approves.

Do not commit cookies, tokens, or `.env` files here.

## What the patch does

Root cause: Cobalt sent Instagram session cookies only to the mobile API (`i.instagram.com/api/v1/media/<id>/info/`), posing as an old Android app. Instagram rejected those web cookies with `challenge_required`, so posts and reels came back as `fetch.empty` even when the cookies were valid. A second bug in the `hasData` check treated a failed GraphQL payload as usable data, which skipped later cookie-backed attempts.

This fork changes three things in `api/src/processing/services/instagram.js`:

1. After the mobile API fails, it requests the web endpoint `https://www.instagram.com/api/v1/media/<id>/info/` with the same cookies. That endpoint accepts web session cookies and returns the media.
2. It derives the numeric media id from the post shortcode when Instagram's oembed/lookup does not return one.
3. It fixes `hasData` so a failed GraphQL reply no longer blocks the cookie path. A payload is treated as usable only when it has media (`gql_data.shortcode_media` or `gql_data.xdt_shortcode_media`), or when it has no `gql_data` key at all (the mobile/web API item shape).

Related upstream work: [imputnet/cobalt#1619](https://github.com/imputnet/cobalt/pull/1619) covers the web API fallback for some cases. This fork also includes the shortcode-to-media-id path and the `hasData` fix.

Verified locally on Cobalt 11.7.1 (upstream `a636575`): 224 of 235 public Instagram reels and posts succeeded (95%).

### Stories

Root cause: Cobalt's story route first resolves the username to a user id through `www.instagram.com/api/v1/users/web_profile_info/`. With our session cookies that endpoint answers `429` with an empty body on every request, so `getStory` stopped with `fetch.empty` before it ever asked for the story. When the lookup did succeed, a missing `reels_media` reply crashed on `media.items` and also surfaced as `fetch.empty`.

The fork adds two things:

4. A story fast path. The number in `/stories/<username>/<id>` is the story's media pk, and `www.instagram.com/api/v1/media/<id>/info/` with web cookies, `x-ig-app-id: 936619743392459` and `x-csrftoken` returns the story item directly. No username lookup is needed. The old `web_profile_info` + GraphQL `reels_media` route stays as a fallback and no longer crashes when it gets no reply.
5. Account rotation on flagged sessions. If the web media endpoint answers `checkpoint_required`, `challenge_required` or `login_required`, the request is retried with up to two other accounts from the cookie pool. This applies to posts, reels and stories.

Expired stories (older than 24 hours) and stories from private accounts the session cannot see return `Media not found or unavailable` from Instagram. Cobalt still reports those as `fetch.empty`.

Keep the cookie pool healthy: an account that returns `checkpoint_required` needs to be verified in a browser by its owner or removed from `cookies.json`.

## Syncing from upstream

```bash
git remote add upstream https://github.com/imputnet/cobalt.git   # once
git fetch upstream
git checkout aisaver/instagram-fix
git rebase upstream/main
# If instagram.js conflicts, keep the web API fallback with account rotation, shortcode helper, hasData fix, and story fast path, then:
git push --force-with-lease origin aisaver/instagram-fix
```

A push to `aisaver/instagram-fix` rebuilds and publishes `ghcr.io/insolowetrust/cobalt:aisaver`.

## Environment

Set these when running the image. Defaults are too tight for a full QA pass (Cobalt allows 20 API requests per minute unless raised), and Instagram needs the cookie file.

| Variable | Suggested value | Why |
| --- | --- | --- |
| `API_URL` | Public URL of this instance, trailing slash | Required by Cobalt for tunneling |
| `COOKIE_PATH` | `/cookies.json` | Path *inside the container* to a Cobalt-format cookies file. Mount the file; do not bake it into the image. |
| `RATELIMIT_MAX` | `100000` | API request cap per window. Default is `20`. |
| `TUNNEL_RATELIMIT_MAX` | `100000` | Tunnel/stream cap per window. Default is `40`. |

## Replacing the stock image

Image: `ghcr.io/insolowetrust/cobalt:aisaver`

It is built from the upstream `Dockerfile` on every push to `aisaver/instagram-fix` (linux/amd64). If the package is private, pull it with `docker login ghcr.io` using a GitHub account that can read packages from `insolowetrust`.

### `docker run`

```bash
docker run --init --read-only --restart unless-stopped \
  --name cobalt -p 9000:9000 \
  -e API_URL=https://your-instance.example/ \
  -e COOKIE_PATH=/cookies.json \
  -e RATELIMIT_MAX=100000 \
  -e TUNNEL_RATELIMIT_MAX=100000 \
  -v /path/to/cookies.json:/cookies.json:ro \
  ghcr.io/insolowetrust/cobalt:aisaver
```

### Docker Compose

Drop this in place of `image: ghcr.io/imputnet/cobalt:11` (or another stock tag):

```yaml
services:
  cobalt:
    image: ghcr.io/insolowetrust/cobalt:aisaver
    init: true
    read_only: true
    restart: unless-stopped
    container_name: cobalt
    ports:
      - 9000:9000/tcp
    environment:
      API_URL: "https://your-instance.example/"
      COOKIE_PATH: "/cookies.json"
      RATELIMIT_MAX: "100000"
      TUNNEL_RATELIMIT_MAX: "100000"
    volumes:
      - ./cookies.json:/cookies.json:ro
```

Keep `cookies.json` on the host only. After a new image is published, pull and recreate the container; do not deploy until Maksym approves.
