# devsecopcicds3

A small practice project: a static marketing site for a fictional company ("CloudStudy Ltd") that
deploys itself to an Amazon S3 static website every time I push to `main`, using GitHub Actions.

**Live site:** http://devsecopcicds3.s3-website.ca-central-1.amazonaws.com/index.html

The goal was not the website. The goal was to build the smallest possible CI/CD pipeline end to end
and understand every piece of it, rather than copying a template I could not explain.

## What's in the repo

| File | Purpose |
|---|---|
| `index.html` | Home page: hero, stats band, services grid, testimonial |
| `about.html` | Second page, mainly to prove multi-page routing works on S3 |
| `error.html` | Custom 404, wired up as the bucket's error document |
| `style.css` | All styling, hand-written, no framework |
| `logo.png` | Favicon and nav logo |
| `.github/workflows/s3.yml` | The deployment pipeline |

## The pipeline

[.github/workflows/s3.yml](.github/workflows/s3.yml) is deliberately three steps:

1. **Checkout** the repo onto the runner.
2. **Configure AWS credentials** from GitHub repository secrets.
3. **Sync** the working directory to the bucket with `aws s3 sync ... --delete`.

```yaml
aws s3 sync . s3://${{ secrets.S3_BUCKET_NAME }} --delete \
  --exclude ".git/*" --exclude ".github/*" \
  --exclude "README.md" --exclude "LICENSE" --exclude "s3.yml"
```

Four secrets drive it, all set in **Settings → Secrets and variables → Actions**:
`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `S3_BUCKET_NAME`.

## What I learnt

**Repository secrets are the whole point.** My first instinct was to put the access key in the
workflow file so I could see it working. That would have published a live AWS credential to a public
repo. Secrets are referenced as `${{ secrets.NAME }}`, never printed in logs, and the workflow file
stays safe to read by anyone. Putting the bucket name and region in secrets too was arguably overkill
(neither is sensitive), but it meant nothing about my account is hard-coded in a public file.

**`aws s3 sync` is not `aws s3 cp`.** `sync` compares what is already in the bucket and only uploads
what changed, which makes repeat deploys fast. Adding `--delete` makes the bucket a mirror of the
repo: delete a file in git and it disappears from the site. Without `--delete` the bucket slowly
fills with files I removed months ago and can no longer account for.

**Excludes matter more than I expected.** Without them, `sync` happily uploads `.git/` to a
publicly readable bucket. That directory contains my full commit history and remote URLs. This was
the single most important thing I got out of the exercise: a deploy step is only as safe as the
things you told it *not* to copy.

**S3 static website hosting is three separate settings, not one.** Turning it on was not enough. The
site only worked once I had:
- Static website hosting enabled, with `index.html` as the index document and `error.html` as the
  error document,
- Block Public Access turned off for the bucket,
- A bucket policy granting `s3:GetObject` on `arn:aws:s3:::devsecopcicds3/*` to `"Principal": "*"`.

Miss any one of those and the symptom is the same generic 403, which is what made it a useful thing
to debug. Public read on objects and public access block are two different gates.

**The website endpoint is not the REST endpoint.** `devsecopcicds3.s3-website.ca-central-1.amazonaws.com`
serves index documents and custom error pages. `devsecopcicds3.s3.ca-central-1.amazonaws.com` does
not: it hands back XML errors and will not resolve `/` to `index.html`. Two URLs, same bucket,
different behaviour.

**HTTP only.** The S3 website endpoint does not do HTTPS, which is why the browser shows "Not
secure". That is a property of the endpoint, not a misconfiguration on my side, and the fix is to
put CloudFront in front of it.

**Pinning action versions.** `actions/checkout@v7` and `aws-actions/configure-aws-credentials@v6.3.0`
are pinned rather than left floating, so a new major release upstream cannot break my deploy without
me choosing to upgrade.

## Results

- Push to `main` publishes the site with no manual steps. Typical run finishes in well under a minute.
- Both pages and the custom 404 serve correctly from the website endpoint.
- Deleting a file in git removes it from the bucket on the next push, so the repo is the single
  source of truth for what is live.
- No credentials anywhere in the repo.

## Known gaps / next steps

These are real limitations of the current setup, not oversights I plan to leave:

- **Long-lived IAM access keys.** The correct pattern is GitHub's OIDC provider assuming an IAM role,
  so no static keys exist at all. This is the first thing I want to replace.
- **The IAM user is broader than it needs to be.** It should be scoped to `s3:PutObject`,
  `s3:DeleteObject` and `s3:ListBucket` on this one bucket.
- **No HTTPS and no CDN.** CloudFront in front of the bucket, with the bucket itself made private and
  reached through Origin Access Control, would fix both at once. (The footer text on the site already
  claims CloudFront is in place. It is not yet.)
- **No cache invalidation step.** Once CloudFront is added, the workflow needs a
  `create-invalidation` call or pushes will not be visible to anyone with a warm cache.
- **No checks before deploy.** There is nothing stopping a broken HTML file from going straight to
  production. An HTML validator or link check as a gating job is the obvious next addition, and is
  the "sec" and "test" part of DevSecOps that this project has not covered yet.

## Running it yourself

1. Create an S3 bucket, enable static website hosting, set index and error documents.
2. Disable Block Public Access and attach a public `s3:GetObject` bucket policy.
3. Create an IAM user with write access to that bucket and generate an access key.
4. Add the four secrets listed above to the repo.
5. Push to `main`.
