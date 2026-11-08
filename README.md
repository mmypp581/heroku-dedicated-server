# Heroku alternative: where each app shape should go, what a real month costs, and when a $79 bare-metal box wins (plan table + migration checklist)

Heroku moved to a sustaining-engineering model in February 2026. Existing apps stay supported, current customers keep their pricing, and Enterprise Account contracts are no longer offered to new customers. Nothing you run today breaks, but the platform has stopped growing, so the practical question is where the next few years of your app should live.

The answer depends on what your app is made of. A Rails monolith with a worker and a Postgres add-on needs a process platform. A Next.js frontend with a handful of API routes needs a function platform. A steady, memory-hungry workload stops fitting either one, and that is the point where a dedicated server becomes the cheapest line item on the page.

> **Quick pick.** Most readers should land on Render: a $7/month web service plus $6/month managed Postgres, using the same web/worker/cron vocabulary you already know. If your Heroku bill is already past roughly $80/month because of memory or worker count, SpinServers' Dallas E3-1280 v5 (4 cores / 8 threads, 32 GB ECC RAM, 1 TB NVMe, 30 TB transfer on a 10 Gbps port) is $79/month flat, and you take on the OS and database yourself.
> [👉 See SpinServers' current dedicated server plans and pricing](https://bit.ly/Spinservers)

## The four SpinServers plans

SpinServers sells bare metal in Dallas and San Jose on monthly terms, with no setup fee on the three packaged tiers. These are the plans on its pricing page as of late September 2026.

| Plan | Core configuration | Price | Billing | Order |
| --- | --- | --- | --- | --- |
| Intel Xeon E3-1280 v5 | 4C/8T @ 3.7 GHz, 32 GB DDR4 ECC, 1 TB M.2 NVMe, 30 TB transfer, 10 Gbps | $79/mo | Monthly | [ Check availability](https://bit.ly/Spinservers) |
| Starter Package | 32 GB RAM, 1 TB NVMe, 10 TB transfer, 10 Gbps, full root | $99/mo | Month-to-month, no setup fee | [ Check availability](https://bit.ly/Spinservers) |
| Growth Package | 256 GB RAM, 2 × 1.6 TB SSD, 20 TB transfer, 10 Gbps, full root | $149/mo | Month-to-month, no setup fee | [ Check availability](https://bit.ly/Spinservers) |
| High Performance Package | 768 GB RAM, 8 × 2 TB SSD, 40 TB transfer, 10 Gbps, full root | $679/mo | Month-to-month, no setup fee | [ Check availability](https://bit.ly/Spinservers) |

San Jose runs different hardware from Dallas, including 16- and 32-core machines with up to 512 GB of RAM, at prices that aren't published on the same page. Availability moves the number too, so the order screen is the only place the figure is final. There is no free plan and no trial, and month-to-month service generally comes without a refund, which makes your first invoice the trial.

The Starter Package is worth a second look for a different reason than the $79 box. It's the same 32 GB and 1 TB NVMe with a smaller 10 TB transfer allowance and a $99 price, so unless it carries newer silicon than the E3-1280 v5, the cheaper machine is the better buy. Ask before ordering.

## What your Heroku app decomposes into

Every Heroku app breaks into the same handful of pieces, and each piece has a named destination on the platforms built around Heroku's model.

| Heroku piece | Render | Fly.io |
| --- | --- | --- |
| Web dyno | Web service | A process group in `fly.toml` |
| Worker dyno | Background worker | Its own process group, scaled separately |
| Release phase | Pre-deploy command | `release_command` under `[deploy]` |
| Procfile | `render.yaml` service definitions | `[processes]` section |
| Eco-style sleep | Free tier spins down after 15 minutes idle | `auto_stop_machines` / `auto_start_machines` |
| Review apps | Preview environments | `superfly/fly-pr-review-apps` GitHub Action |
| Postgres add-on | Managed Postgres from $6/month | Managed Postgres from $38/month |

The web process moves first and moves easily: build on the new platform, inject your config vars, verify through a temporary URL. Workers are a config change. Postgres is the piece that decides whether your cutover takes a Tuesday afternoon or a weekend, and it's covered further down.

Fly.io earns its place on this list for a different reason than price. It runs each Machine as a Firecracker microVM with its own kernel on bare metal in 17 regions, so hardware isolation is the default rather than an enterprise tier. That matters if you're hosting other people's code. It also bills a stopped Machine only for its root filesystem at $0.15/GB over 30 days, which is the closest thing to a Heroku Eco dyno that still answers the first request immediately.

## What a month actually costs

Heroku's list prices haven't moved. Eco is $5/month for 512 MB that sleeps after 30 minutes idle, Basic is $7/month for 512 MB always on, Standard-1X is $25/month for 512 MB, and Standard-2X is $50/month for 1 GB. The free tier that made Heroku the default for side projects disappeared in November 2022 and was replaced by the $5 Eco dyno.

The bill gets uncomfortable when you add what production needs. A minimal stack of one Standard-1X dyno, a Postgres instance and a Redis add-on is commonly estimated at around $64/month before autoscaling and any other add-on. That figure comes from a competing platform's blog, so treat it as an order of magnitude rather than an invoice. Heroku's entry add-on tiers start lower (Postgres Essential-0 at $5/month, Key-Value Store at $3/month for 25 MB) and climb once you need a production tier.

| Setup | What you get | Monthly |
| --- | --- | --- |
| Heroku Basic | 512 MB dyno, always on | $7 |
| Heroku Standard-1X | 512 MB dyno | $25 |
| Heroku Standard-2X | 1 GB dyno | $50 |
| Heroku minimal production shape | Standard-1X + Postgres + Redis, no autoscale | ~$64 (estimate) |
| Render | 0.5 CPU / 512 MB web service + 256 MB Postgres | $7 + $6 |
| Fly.io | shared-cpu-1x 1 GB Machine + Managed Postgres | $6.95 + from $38 |
| DigitalOcean App Platform | entry-level container | from $5 |
| Railway | 1 vCPU / 1 GB always on, Pro workspace | ~$30 metered; $20 fee includes $20 of usage |
| SpinServers E3-1280 v5 | 4C/8T, 32 GB RAM, 1 TB NVMe, 30 TB transfer | $79 |
| SpinServers Growth | 256 GB RAM, 2 × 1.6 TB SSD, 20 TB transfer | $149 |

Read that table by shape, not by headline. Render's $13 combination is the cheapest honest replacement for the Heroku starter pattern, and it will stay cheaper than a dedicated box until you need more than about 2 GB of RAM or a lot of worker concurrency. Fly.io's compute is the cheapest always-on 1 GB Machine on the list; its Managed Postgres starts at $38/month, which is a production cluster rather than a development tier with a downtime allowance, and it's available in fewer regions than Fly's compute with no multi-region read replicas.

Railway meters vCPU and memory per second, which reads cheap until a runaway worker or a traffic spike moves a $12 invoice to $40 in one cycle. DigitalOcean App Platform starts at $5/month, carries a 99.95% monthly SLA with service credits between 10% and 100% of monthly fees, and keeps droplets and managed databases in the same account. Vercel at $20/month is excellent for function-shaped frontends and wrong for a Rails or Django monolith with Sidekiq or Celery, because functions start per request and stop when they return. Netlify at $9/month and Upsun at $10/month cover narrower slices of the same ground, and Encore keeps push-to-deploy while provisioning into your own AWS or GCP account. AWS Elastic Beanstalk, Google App Engine (around $0.0579/hour) and Azure App Service (from $13.14/month) exist for teams already committed to those clouds.

## When a dedicated server is the cheaper answer

Per-gigabyte memory pricing is where PaaS bills break. Heroku's largest standard dyno is 1 GB for $50/month; a workload that genuinely holds 32 GB resident doesn't scale horizontally into 32 dynos, it just needs a bigger machine. That's the shape SpinServers sells: 32 GB of ECC RAM with a 1 TB NVMe drive and 30 TB of transfer for $79/month, or 256 GB of RAM for $149.

What comes with it: full root access, IPMI or iKVM remote management, a VPC with private Layer 2 connectivity, a DDoS-protected multihomed network, dedicated IPv4 (and IPv6 /64 on listed configurations), instant provisioning, and automated Linux, Windows or ESXi installs. Facilities are advertised as PCI compliant, but the compliance work on your server stays yours. Spin Servers suits teams that do their own scanning and evidence collection better than teams shopping for managed PCI.

What doesn't come with it: a build pipeline, a managed database, backups, monitoring, or anyone on call. You close that gap with software. Dokku gives you Heroku buildpacks, `git push` deploys, a Postgres plugin that creates the service and wires `DATABASE_URL` into your app, and per-process scaling so web and worker run as separate containers. Coolify gives you a UI, deployment from a repository, Dockerfile, image or Compose file, and automatic domain and HTTPS setup. Both leave you maintaining the servers, and Dokku is single-host by default, so a dead machine takes the app and its database down together.

[👉 Check the $79 Dallas box and its current availability](https://bit.ly/Spinservers)

## Who should not buy a dedicated server

Skip it if any of these describe you.

- Your Heroku bill is under about $79/month. You'd pay more for the privilege of doing your own patching.
- You want managed Postgres with backups and point-in-time recovery without building it. Render's $6 tier or Heroku's own add-on does that for less than a server.
- You rely on autoscaling for traffic you can't predict. A single box scales vertically and then stops.
- You need multi-region failover on day one.
- You need managed PCI compliance or someone else to answer for the compliance posture.
- You expect a refund if the first month goes badly. Assume the opposite and plan accordingly.

The Growth Package at 256 GB of RAM is the one most readers should not buy either. It exists for in-memory databases, large caches and dense virtualization, and if your app fits in 32 GB, the extra $70/month buys nothing you'll notice. Start on the $79 machine and move up when a metric tells you to.

## The migration mistakes that cost the most

The web process is not where migrations fail. Databases and in-flight jobs are, because both keep changing while the old app is still serving traffic.

For a small database, a maintenance window with a final export and restore is enough. A larger or busier one needs replication plus a short write pause to catch the tail. The trap comes after cutover: once the new database accepts writes, pointing DNS back at the old one does not carry those writes with it. Reverting is a reconciliation job, not a DNS change, so decide the rollback plan before the window opens, not during it.

After the switch, run one real business flow end to end. Submit a request, confirm the row landed in the new database, confirm the job it queued actually completed. Keep the old deployment alive until the new one has served real traffic and the backlog of leftover background jobs has drained.

You don't have to do it in one jump. Leave production on Heroku and run a staging clone on the candidate platform for a week, then compare error rates and latency before touching DNS. Model the cost at steady state and at three times your current traffic, because the number that surprises people is the second one.

[👉 Look at the 256 GB Growth Package if your dataset needs it](https://bit.ly/Spinservers)

## Rules for picking

- One dyno, one database, predictable traffic → Render at $13/month for the pair.
- Global latency matters and your team is comfortable with a CLI → Fly.io, $6.95/month for an always-on 1 GB Machine, budget $38+ for managed Postgres.
- You want a cloud account you already have → DigitalOcean App Platform from $5, or Encore provisioning into your own AWS or GCP.
- Frontend and functions only → Vercel or Netlify, with the backend somewhere else.
- Steady memory or CPU load above roughly $80/month of PaaS spend, and you'll run the box → SpinServers at $79/month plus Dokku or Coolify.

Before you pay SpinServers, confirm three things on the order screen: the exact CPU model and RAM for the location you picked, whether the plan is billed monthly with no setup fee, and the transfer allowance, since the $79 and $99 tiers differ by 20 TB there. Then confirm what your rollback looks like, because the machine arrives in minutes and the migration plan is the part that takes a week.

[👉 Start with the $79 entry server and size up when a metric says so](https://bit.ly/Spinservers)
