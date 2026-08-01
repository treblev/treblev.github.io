---
layout: post
title: "What Happens When a Telegram Image Reaches Groundhog"
date: 2026-07-31
categories: groundhog
---

Telegram image uploads look immediate from the chat, but they involve two
separate paths: importing the image into Groundhog and sending a message back
to Telegram. Keeping those paths distinct is useful, but it also means their
timing must be handled carefully.

## The upload path, step by step

1. **You send a screenshot in Telegram.** OpenClaw receives the image file and
   its caption, if you included one.

2. **The direct upload handler starts the import.** It passes that exact image
   to Groundhog right away. Its only job is to return the metrics from that
   image, such as date, distance, duration, pace, and heart rate.

3. **Groundhog's local vision model reads the screenshot.** The model extracts
   only the visible workout facts. Groundhog uses the date shown in the image;
   a caption can supply a date only when the image date is unreadable.

4. **Groundhog saves the result in DuckDB.** The activity is upserted, so
   retrying the same screenshot does not create a duplicate record. It also
   records an `upload_imported` event as an audit trail for future summaries.

5. **The direct handler replies about that same image.** This response is
   immediate and tied to the attachment currently being processed.

6. **The periodic watcher is a safety net.** Once each minute it looks for new
   images the direct handler may have missed, imports them, and records their
   results. It does not send a later chat reply.

The outbox is intentionally separate. It is for operational notifications such
as stock alerts and failed jobs, which can safely be delivered later because
they are not replies to a particular Telegram image.

## The important distinction

The image-import path is tied to one specific attachment. Groundhog reads that
file with the local vision model, extracts its visible metrics, saves the
activity, and the direct handler can respond with those exact metrics.

The outbox path is different. It is intentionally asynchronous: a background
job later sends durable operational notifications such as stock alerts. It
does not know which image message is currently open in the Telegram chat, and
it cannot reply to an individual attachment.

## Why an old activity summary appeared beside a new image

Previously, successful image imports also added an “Imported activity” message
to the outbox. Vision extraction can take minutes. If an older image completed
after a newer one had arrived, the background delivery job could send the
older message near the newer image. The extracted facts were correct, but the
placement made the message look like it described the wrong screenshot.

## The fix

Image imports still create a durable `upload_imported` event, so Groundhog has
an audit trail and daily summaries can include the import. They no longer add
a background Telegram message to the outbox.

That leaves each channel with one clear responsibility:

- The direct upload handler replies only about the image it just imported.
- The periodic watcher silently imports any attachment the direct handler did
  not process.
- The outbox remains for notifications that are genuinely independent of a
  particular chat message, such as stock alerts and job failures.

The result is a more trustworthy conversation: no delayed activity result can
masquerade as the reply to a later screenshot.
