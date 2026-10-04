---
theme: seriph
title: "CSS4005 — Lecture 5: AWS Storage (EBS and S3)"
info: |
  CSS4005 — Cloud Infrastructure Construction · Narxoz University
  Lecture 5 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# CSS4005 — Cloud Infrastructure Construction

## Lecture 5: AWS Storage (EBS and S3)

<div class="pt-8 opacity-70">
Nauruzbayeva Farikha · Lesson 5
</div>

---
layout: default
---

# Recap — Lesson 4 (AWS Networking: VPC)

<v-clicks>

- What makes a subnet "public"? <span v-click class="opacity-60">(its route table has a `0.0.0.0/0` route to an Internet Gateway)</span>
- A private App server needs to download updates but must not accept connections from the internet. What do you add? <span v-click class="opacity-60">(a NAT Gateway in a public subnet + a `0.0.0.0/0 → NAT` route in the private route table)</span>
- How do you allow DB:3306 only from the App tier? <span v-click class="opacity-60">(SG-DB inbound rule: TCP 3306, source = SG-APP — reference the security group, not an IP)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] Three kinds of storage: block, file, object
- [ ] EBS: volumes, types, attach → format → mount
- [ ] EBS snapshots & encryption
- [ ] S3: buckets, objects, keys
- [ ] S3: storage classes, lifecycle, versioning
- [ ] S3: public access, website hosting, presigned URLs
- [ ] Choosing storage + not paying for forgotten resources
- [ ] Live demo → straight into today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Last week you built the three-tier VPC. Your EC2 web server is running.
Users start uploading profile photos — you save them to the instance's
disk. One day someone terminates the instance to "resize it", and every
photo is gone. Another day a bug overwrites half the files, and there is
no older copy anywhere.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today is about where data should live in AWS so that it survives
instances, mistakes, and the bill.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## Block, file, and object storage

---
---

# Three ways to store bytes

| | Block | File | Object |
|---|---|---|---|
| You see | A raw disk | A shared folder | Objects behind an HTTP API |
| Accessed via | OS mounts it, formats it | NFS / SMB mount | `GET` / `PUT` over HTTPS |
| Attach to | One instance (usually) | Many instances | Anyone with permission |
| AWS service | **EBS** | **EFS** | **S3** |
| Linux analog | `/dev/sdb` | NFS share | — (a web API, not a disk) |

<div v-click class="mt-6 text-sm opacity-70">
The question is never "which is best" — it's "which shape does this data
have, and who needs to read it?"
</div>

---
---

# Block storage in one sentence

<v-clicks>

- AWS hands your instance an **empty disk** — a sequence of blocks with no
  filesystem on it.
- Your OS does the rest: partition (optional), `mkfs`, `mount`, `/etc/fstab`.
- Exactly what you'd do with a new SSD in a physical server — the
  Linux skills from week 2 apply unchanged.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-blue-500/10 text-sm">
Databases, OS boot disks, anything that needs a real filesystem with low
latency → block storage.
</div>

---
---

# Object storage in one sentence

<v-clicks>

- You store **objects** (a file's bytes + metadata) under a **key**, in a
  **bucket**.
- No mounting, no filesystem, no partial in-place edits — you `PUT` a whole
  object, you `GET` a whole object (or a byte range).
- Practically unlimited capacity, reachable over HTTPS from anywhere you
  allow.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-blue-500/10 text-sm">
User uploads, backups, logs, static website files, build artifacts →
object storage.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## Amazon EBS — Elastic Block Store

---
---

# Instance store vs EBS

| | Instance store | EBS volume |
|---|---|---|
| Physically | Disk inside the host machine | Network-attached volume |
| Survives reboot | Yes | Yes |
| Survives stop / terminate | **No** — data is lost | Yes (unless deleted on termination) |
| Available on | Only some instance types | Every instance type |
| Good for | Caches, scratch space, temp data | Boot disks, databases, app data |

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
Instance store is fast but <b>ephemeral</b>. Never keep the only copy of
anything important there.
</div>

---
---

# An EBS volume lives in one AZ

<v-clicks>

- A volume is created **in a specific Availability Zone** and is replicated
  inside that AZ to protect against a single hardware failure.
- It can only be attached to an instance **in the same AZ**.
- Instance in `eu-central-1a`, volume in `eu-central-1b` → you simply can't
  attach it.
- Moving data to another AZ = snapshot → new volume in the other AZ
  (Block 3).

</v-clicks>

<div v-click class="mt-6 text-sm opacity-70">
Same idea as last week's subnets: a subnet is AZ-scoped, and so is a
volume.
</div>

---
---

# EBS volume types — SSD

| Type | What it's for | Key property |
|---|---|---|
| **gp3** (General Purpose SSD) | Default for almost everything: boot disks, web servers, dev/test, small DBs | Baseline **3,000 IOPS + 125 MiB/s** at any size; IOPS and throughput can be raised separately from size |
| gp2 (older) | Legacy default | IOPS scale with size — prefer gp3 for new volumes |
| **io2** Block Express (Provisioned IOPS SSD) | Large, latency-sensitive databases | Highest IOPS you provision explicitly; designed for 99.999% durability; supports Multi-Attach |

<div v-click class="mt-6 text-sm opacity-70">
IOPS = small random reads/writes per second (databases care).
Throughput = MiB per second (big sequential reads care).
</div>

---
---

# EBS volume types — HDD

| Type | What it's for | Key property |
|---|---|---|
| **st1** (Throughput Optimized HDD) | Big sequential workloads: log processing, data warehouses, streaming | Cheap per GB, good MiB/s, poor random IOPS |
| **sc1** (Cold HDD) | Rarely accessed large data | Lowest cost per GB of all EBS types |

<v-clicks>

- st1 and sc1 **cannot be boot volumes**.
- If you're unsure → **gp3**. Change type later with Elastic Volumes —
  no detach needed.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your instance runs in <code>eu-central-1a</code>. You create a 20 GiB
gp3 volume in <code>eu-central-1b</code>. You click "Attach". What
happens, and how do you fix it?
</div>

<div v-click class="mt-8 text-lg opacity-70">
The instance doesn't show up in the list — EBS volumes attach only within
their own AZ. Delete it and create it in <code>1a</code>, or snapshot it
and restore the snapshot into <code>1a</code>.
</div>

---
---

# Attach → format → mount

```bash {1|2-3|4|5-6|7}
aws ec2 attach-volume --volume-id vol-0abc --instance-id i-0123 --device /dev/sdf
# on the instance:
lsblk                                # new disk appears, e.g. nvme1n1, no mountpoint
sudo file -s /dev/nvme1n1            # "data" → empty, no filesystem yet
sudo mkfs -t xfs /dev/nvme1n1        # ONLY on a new, empty volume
sudo mkdir /data && sudo mount /dev/nvme1n1 /data
df -h /data                          # mounted and usable
```

<div v-click class="mt-4 p-4 rounded bg-blue-500/10 text-sm">
On modern (Nitro) instances a volume attached as <code>/dev/sdf</code>
shows up as <code>/dev/nvme1n1</code>. Always check <code>lsblk</code> —
never guess the device name.
</div>

---
---

# Persist the mount: `/etc/fstab`

```bash
sudo blkid /dev/nvme1n1
# /dev/nvme1n1: UUID="4f2c...e91" TYPE="xfs"
```

```text
# /etc/fstab
UUID=4f2c...e91  /data  xfs  defaults,nofail  0  2
```

<v-clicks>

- Use the **UUID**, not `/dev/nvme1n1` — NVMe device names can change
  between boots.
- `nofail` = boot anyway if the volume is missing. Without it, a detached
  volume can leave the instance stuck at boot, unreachable over SSH.
- Test before rebooting: `sudo umount /data && sudo mount -a`.

</v-clicks>

---
---

# `mkfs` destroys data

<v-clicks>

- `mkfs` writes a **new, empty** filesystem. Run it on a volume restored
  from a snapshot and you've wiped the restore.
- Rule: run `sudo file -s /dev/<dev>` first. Output `data` → empty, format
  it. Output mentions `XFS`/`ext4` → it already has data, **just mount**.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-red-500/10 text-lg font-bold">
Formatting the wrong disk is the EBS equivalent of <code>down -v</code>:
no prompt, no undo.
</div>

---
---

# Delete on termination

<v-clicks>

- The **root** volume created with an instance: `DeleteOnTermination =
  true` by default → terminate the instance, the root disk goes too.
- **Additional** volumes you attach: `false` by default → terminate the
  instance, the volume keeps existing (status `available`).
- An `available` volume is attached to nothing but is still **billed per
  GB-month provisioned**.

</v-clicks>

<div v-click class="mt-6 text-sm opacity-70">
Every semester someone's credits run out because of forgotten
<code>available</code> volumes. Check EC2 → Volumes at the end of every lab.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## Snapshots & encryption

---
---

# EBS snapshots

<v-clicks>

- A **snapshot** is a point-in-time backup of a volume.
- **Incremental**: the first snapshot copies all used blocks; later ones
  store only the blocks that changed since the previous snapshot.
- Stored durably in S3 by AWS — you don't see them in your buckets, you
  manage them under EC2 → Snapshots.
- Snapshots are **regional**: restore one to a new volume in **any AZ** of
  that region, or copy it to another region.

</v-clicks>

---
---

# Snapshot → restore

```bash {1|2|3-4|5}
aws ec2 create-snapshot --volume-id vol-0abc --description "before upgrade"
aws ec2 wait snapshot-completed --snapshot-ids snap-0def
aws ec2 create-volume --snapshot-id snap-0def \
    --availability-zone eu-central-1b --volume-type gp3
# attach the new volume → mount it (do NOT mkfs — it already has your data)
```

<v-clicks>

- Snapshot → volume in another AZ is how you **move** block data between AZs.
- For consistent DB snapshots: flush/stop writes first, or use the
  database's own backup tools.

</v-clicks>

---
---

# EBS encryption

<v-clicks>

- Encrypted with a **KMS** key (AWS-managed `aws/ebs` key by default).
- Covers: data at rest on the volume, data moving between instance and
  volume, and every snapshot made from it.
- Transparent to the OS — no code changes, mount it as usual.
- **Encryption by default** is a per-region account setting: turn it on
  once and every new volume is encrypted.
- An existing unencrypted volume can't be "switched on": snapshot it → copy
  the snapshot with encryption → create a volume from the copy.

</v-clicks>

---
---

# EFS — for contrast

<v-clicks>

- **Amazon EFS** = managed **NFS** file system.
- Mounted by **many** Linux instances at once, across **multiple AZs**.
- Grows and shrinks automatically — billed for what you store, not a
  pre-provisioned size.
- Use it when several servers need the **same files** (shared uploads,
  shared config). Use EBS when one server needs its own fast disk.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your database volume is in <code>eu-central-1a</code>. You need a copy of
that data on a new instance in <code>eu-central-1c</code> for testing.
What are the steps?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Snapshot the volume → create a new volume from the snapshot in
<code>1c</code> → attach it to the test instance → mount it (no
<code>mkfs</code>).
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Amazon S3 — Simple Storage Service

---
---

# Buckets, objects, keys

<v-clicks>

- **Bucket** — a container for objects. The name is **globally unique**
  across all AWS accounts (3–63 chars, lowercase, digits, `-`, `.`).
- A bucket **lives in one region** you choose — the data stays there unless
  you replicate it.
- **Object** — the data (up to 5 TB) + metadata, identified by a **key**.
- Designed for 11 nines (99.999999999%) of durability; strongly consistent
  read-after-write.

</v-clicks>

```text
s3://narxoz-photos-farikha/users/42/avatar.png
     └──── bucket ────┘ └────── key ──────┘
```

---
---

# "Folders" are just key prefixes

<v-clicks>

- S3 has a **flat** namespace. `users/42/avatar.png` is one key with
  slashes in it — not a file inside directories.
- The console *draws* folders by grouping keys on `/`.
- Prefixes still matter: lifecycle rules, permissions, and listings can
  target a prefix like `logs/`.

</v-clicks>

```bash
aws s3 ls s3://narxoz-photos-farikha/users/42/
aws s3 cp avatar.png s3://narxoz-photos-farikha/users/42/avatar.png
aws s3 sync ./site s3://narxoz-site-farikha/
```

---
---

# S3 storage classes

| Class | Use for | Trade-off |
|---|---|---|
| **Standard** | Frequently accessed data (default) | Highest storage price, no retrieval fee |
| **Intelligent-Tiering** | Unknown or changing access patterns | Moves objects between tiers automatically; small monitoring fee |
| **Standard-IA** | Infrequent access, needs fast retrieval | Cheaper storage, per-GB retrieval fee, 30-day minimum |
| **One Zone-IA** | Re-creatable infrequent data | Stored in a single AZ — lost if that AZ is lost |
| **Glacier Instant / Flexible / Deep Archive** | Archives, compliance, long-term backups | Cheapest storage; slower and/or paid retrieval, 90–180 day minimums |

<div v-click class="mt-4 text-sm opacity-70">
Exact prices differ per region and change — use the AWS Pricing
Calculator, not memory.
</div>

---
---

# Lifecycle rules — let S3 move data for you

```json
{
  "Rules": [{
    "ID": "logs-to-ia-then-expire",
    "Filter": { "Prefix": "logs/" },
    "Status": "Enabled",
    "Transitions": [{ "Days": 30, "StorageClass": "STANDARD_IA" }],
    "Expiration": { "Days": 365 }
  }]
}
```

<v-clicks>

- **Transition**: after N days, move objects to a cheaper class.
- **Expiration**: after N days, delete them.
- Set it once, never manually clean up old logs again.

</v-clicks>

---
---

# Versioning

<v-clicks>

- With versioning **enabled**, every overwrite keeps the previous object as
  an older **version** with its own version ID.
- A `DELETE` doesn't erase data — it adds a **delete marker**. Remove the
  marker and the object is back.
- Once enabled, versioning can be **suspended** but never fully turned off.
- Old versions are billed too → pair versioning with a
  `NoncurrentVersionExpiration` lifecycle rule.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Versioning is your protection against "a bug overwrote half the files" from
the scenario.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Versioning is enabled. Someone runs <code>aws s3 rm
s3://bucket/report.pdf</code>. Is the file gone?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No — S3 added a delete marker on top. The previous version still exists;
delete the marker (or copy the old version back) to restore it.
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Who can read your bucket?

---
---

# Private by default

<v-clicks>

- New buckets are **private**: only identities in your account with
  permission can read them.
- **Block Public Access** (BPA) is **on by default** for new buckets — four
  switches that override any policy or ACL trying to make data public.
- New objects are **encrypted at rest by default** (SSE-S3).

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-red-500/10 text-sm">
Most "S3 data leak" headlines are someone turning BPA off and attaching a
public policy to the wrong bucket. Leave BPA on unless you have a specific,
reviewed reason.
</div>

---
---

# Bucket policies — preview

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111122223333:role/app-server" },
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::narxoz-photos-farikha/users/*"
  }]
}
```

<v-clicks>

- A JSON document attached to the bucket: **who** (Principal) can do
  **what** (Action) on **which objects** (Resource).
- Same policy language as IAM — **next week's** whole topic.

</v-clicks>

---
---

# Static website hosting

<v-clicks>

- S3 can serve a static site (`index.html`, CSS, JS, images) from a
  **website endpoint**.
- Requires: enable website hosting, set an index document, and allow public
  read (BPA off for that bucket + a public-read policy).
- The S3 website endpoint is **HTTP only**. For HTTPS and a custom domain,
  put **CloudFront** in front — and then the bucket can stay private.

</v-clicks>

<div v-click class="mt-6 text-sm opacity-70">
Fine for a lab demo or a portfolio page. For anything real: CloudFront +
private bucket.
</div>

---
---

# Presigned URLs — temporary access

```bash
aws s3 presign s3://narxoz-photos-farikha/users/42/avatar.png --expires-in 300
# https://narxoz-photos-farikha.s3.eu-central-1.amazonaws.com/users/42/avatar.png?X-Amz-...
```

<v-clicks>

- A URL that carries a signature made with **your** credentials, valid for
  a limited time.
- Anyone holding it can `GET` that one object until it expires — the bucket
  stays private.
- Typical use: the app generates a short-lived download link for a
  logged-in user.
- Maximum lifetime is 7 days, and it stops working early if the signing
  credentials expire.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 6
## Choosing storage & cleaning up

---
---

# Which storage for which workload?

| Workload | Choice | Why |
|---|---|---|
| OS boot disk | EBS gp3 | Block device, survives stop |
| PostgreSQL on EC2 | EBS gp3 (io2 if very IOPS-heavy) | Low-latency block storage |
| Shared uploads for 3 web servers | EFS — or better, S3 | Many readers/writers |
| User photos, backups | S3 Standard + versioning | Durable, cheap, HTTP-accessible |
| Logs kept 1 year | S3 + lifecycle → IA → expire | Pay less as data cools |
| Temporary scratch / cache | Instance store | Fast, and losing it is OK |

---
---

# How storage is billed

<v-clicks>

- **EBS**: per GB-month **provisioned** — a 100 GiB volume with 1 GiB of
  data is billed as 100 GiB. Plus snapshot storage. Plus extra IOPS /
  throughput above gp3 baseline.
- **S3**: per GB-month **actually stored** (every version counts) + per
  request + data transferred out to the internet.
- **EFS**: per GB-month stored.

</v-clicks>

<div v-click class="mt-6 text-sm opacity-70">
No numbers on purpose — they vary by region and change. Estimate with the
AWS Pricing Calculator before you build.
</div>

---
---

# End-of-lab cleanup checklist

<v-clicks>

- [ ] EC2 → Volumes: no forgotten volumes in state `available`
- [ ] EC2 → Snapshots: delete lab snapshots you don't need
- [ ] S3: empty the bucket — **including old versions and delete markers** —
      then delete it
- [ ] Stop or terminate lab instances

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
A versioned bucket that "looks empty" in the console may still hold every
old version. Toggle <b>Show versions</b> before you call it clean.
</div>

---
layout: section
transition: slide-left
---

# Block 7
## Today's practice

---
---

# Today's practice — a disk and a bucket that survive mistakes

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Part 1 — block storage**

1. Create a volume (EBS, or a loop-device file on Linux)
2. Format it, mount it, persist it in `/etc/fstab` with `nofail`
3. Snapshot it, "lose" a file, restore from the snapshot

**Part 2 — object storage**

4. Bucket with versioning; overwrite a file, restore the old version
5. Lifecycle rule, Block Public Access, presigned URL

</div>
<div>

```bash
lsblk
sudo file -s /dev/nvme1n1
sudo mkfs -t xfs /dev/nvme1n1
sudo blkid /dev/nvme1n1
aws s3api put-bucket-versioning \
  --bucket <name> \
  --versioning-configuration Status=Enabled
aws s3 presign s3://<name>/notes.txt
```

<div v-click class="mt-4 opacity-70">
No AWS account? Track B does the same with Linux loop devices and a local
S3-compatible server. Submit <code>REPORT.md</code> with commands +
screenshots.
</div>

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Choose block, file, or object storage for a given workload
- [ ] Create, attach, format, and persistently mount an EBS volume in the
      correct AZ
- [ ] Back up and restore a volume with snapshots, and explain why they're
      incremental
- [ ] Use S3 versioning, lifecycle rules, Block Public Access, and
      presigned URLs
- [ ] Find and delete the storage resources that keep costing money

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice and submit `REPORT.md` if you didn't finish
      in class
- [ ] Run the cleanup checklist — volumes, snapshots, versioned buckets
- [ ] Re-read the bucket policy JSON from today: next week we take that
      policy language apart

---
layout: end
---

# Next lecture

AWS IAM and Cloud Security — users, roles, policies, and least privilege.
