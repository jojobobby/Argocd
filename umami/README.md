# Umami - web analytics for the Cosmic shop

First-party, cookieless product analytics (https://umami.is): page views, referrers,
countries, devices, custom events (buttons clicked), funnels, journeys and retention.
Chosen over Prometheus/Grafana for this because the question is *what do visitors look at
and where do they drop off*, not *is the service up*. Chosen over hosted analytics because
the shop's privacy policy promises no third-party trackers and no analytics cookies, and
Umami keeps both promises: it identifies a visitor for one day by a salted hash and stores
nothing that names them.

- **URL**: https://analytics.cosmicv2.com (DNS A record -> 147.135.8.100, like the other
  `cosmicv2.com` hosts; created out of band at OVH)
- **ArgoCD app**: `umami` (`Arcana-Argocd-Apps/prod/umami.yaml`), namespace `umami`
- **Database**: CloudNativePG cluster `umami-db` in the same namespace (5Gi, one instance).
  The operator writes the connection string to Secret `umami-db-app`, key `uri`; that is
  Umami's `DATABASE_URL`. No other secret is needed - Umami derives its signing secret
  from the database URL when `APP_SECRET` is unset.

## First login

Umami ships with the user `admin` / password `umami`. On first deploy the admin
password is rotated straight away (through the API, to a random value) and stored in
Secret `umami/umami-admin` (keys `username`, `password`). Read it with:

```powershell
kubectl -n umami get secret umami-admin -o jsonpath='{.data.password}' | % { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

Change it from Profile > Change password once logged in if you prefer your own.

## Wiring a site up

1. Settings > Websites > Add website. Name it after the host (`shop-dev.cosmicv2.com`,
   `shop.cosmicv2.com`) and copy the **Website ID**.
2. The Cosmic shop reads two settings, both env-overridable (Helm `website.analytics`
   in the Cosmic chart): `UMAMI_SCRIPT_URL` = `https://analytics.cosmicv2.com/stats.js`
   and `UMAMI_WEBSITE_ID` = that id. With both set the layout emits the tracker tag and
   widens the page's Content-Security-Policy to the analytics origin; with either empty
   nothing is emitted.
3. Buttons and links carry `data-umami-event="..."` (plus `data-umami-event-sku` where
   there is one), so clicks show up under Events with no JavaScript of our own.

The tracker is served as `/stats.js` and posts to `/api/collect` (see values.yaml);
the defaults `/script.js` and `/api/send` sit on the common ad-blocker lists.

## Reading the numbers

Dashboard per website: pages, referrers, countries, browsers, devices, events.
Reports > Funnel: e.g. `/Catalog` -> `/Item/*` -> `/Checkout/*` -> `/Success`.
Reports > Journey: the paths visitors actually take. Visits arriving from the in-game
"Buy Gold" button land on `/?src=game`, so the query parameter report separates
game-driven traffic from direct visits.

## Upgrading

GHCR only publishes `postgresql-latest` for Umami v3, so the image is pinned by digest in
values.yaml. To move to a new release, resolve the current digest of `postgresql-latest`
(the OCI index digest; `docker manifest inspect ghcr.io/umami-software/umami:postgresql-latest -v`
or a HEAD on the registry's manifests endpoint), check its build date against the release
on https://github.com/umami-software/umami/releases, then set `image.digest` and
`appVersion` in Chart.yaml. Umami migrates its schema itself on startup; the Deployment
uses `Recreate` so only one version ever migrates at a time.
