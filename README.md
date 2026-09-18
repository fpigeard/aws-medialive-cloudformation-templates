# AWS MediaLive CloudFormation Templates

Ready-to-use AWS CloudFormation templates for live streaming workflows using AWS Elemental MediaLive, AWS Elemental MediaPackage v2, and Amazon CloudFront.

## Templates

| Template | Codec | Rate Control | Resiliency | Latency |
|---|---|---|---|---|
| [Low-Latency HLS](CloudFormation_Dual-QVBR_LL-HLS_EML-H264_EMPv2_CloudFront.yaml) | H.264 | QVBR | Standard |Low-Latency |
| [Single H264](CloudFormation_Single_EML-H264_EMPv2_CloudFront.yaml) | H.264 | CBR | Single-pipeline | Standard-Latancy |
| [Dual H264 CBR](CloudFormation_Dual-CBR_EML-H264_EMPv2_CloudFront.yaml) | H.264 | CBR | Standard | Standard-Latancy |
| [Dual H264 QVBR](CloudFormation_Dual-QVBR_EML-H264_EMPv2_CloudFront.yaml) | H.264 | QVBR | Standard | Standard-Latancy |
| [HEVC](CloudFormation_EML-HEVC_MediaPackagev2_CloudFront.yaml) | H.265 (HEVC) | CBR | Single-pipeline | Standard-Latancy |
| [Single H264 Vertical](CloudFormation_Single_EML-H264_Vertical_EMPv2_CloudFront.yaml) | H.264 | CBR | Single-pipeline | Standard-Latancy |

---

## Low-Latency HLS (LL-HLS) Template

The HTTP Live Streaming (HLS) protocol allows delivery of live streams to global-scale audiences. Historically, HLS has favored stream reliability over latency. Low-Latency HLS (LL-HLS) is an extension of the protocol that appeared in 2020 enabling low-latency video streaming while maintaining scalability. It allows reduction of live streaming latency by a factor of two. While regular HLS latency usually ranges between 12 and 30 seconds depending on the workflow configuration and the player capabilities, LL-HLS brings end-to-end workflows latency to between 5 and 10 seconds.

The typical use cases for a low-latency OTT workflow is for live streaming scenarios such as sport events, live betting, interactive events, and breaking news where minimizing the delay between the live action and the viewer is critical to maintain real-time engagement and avoid spoilers. This brings streaming latency much closer to traditional broadcast latency, which averages around 6 seconds, and helps prevent viewers of live sports streams from being spoiled by nearby televisions or by updates on social media from the stadium or broadcast audiences.

In 2024, AWS Elemental MediaPackage introduced support for packaging media streams using Low-Latency HLS (LL-HLS), with both Transport Stream (TS) and CMAF segments.

This demo post provides a ready-to-use proof of concept (POC) that customers can quickly deploy using the provided AWS CloudFormation template. The template automatically provisions a test environment with all the required media services preconfigured to build a complete LL-HLS streaming workflow.

The architecture leverages multiple AWS media services—namely AWS Elemental MediaLive, AWS Elemental MediaPackage, and Amazon CloudFront—to enable an end-to-end low-latency streaming workflow based on LL-HLS.

### Architecture

```
Input Source (S3) → MediaLive (H.264 QVBR, 1s GOP) → MediaPackage v2 (CMAF, LL-HLS) → CloudFront (HTTP/2+3) → Player
```

### LL-HLS Configuration Highlights

- **MediaLive**: GOP size 1 second, segment length 1 second, QVBR rate control
- **MediaPackage v2**: LL-HLS manifest with `ProgramDateTimeIntervalSeconds: 1`
- **CloudFront**: HTTP/2and3, cache policy with `_HLS_msn` and `_HLS_part` query strings, CORS response headers policy

### Deployment

```bash
aws cloudformation create-stack \
  --stack-name my-ll-hls-workflow \
  --template-body file://CloudFormation_Dual-QVBR_LL-HLS_EML-H264_EMPv2_CloudFront.yaml \
  --capabilities CAPABILITY_IAM \
  --region eu-west-1
```

---

## Single H264 Vertical Template

Vertical video (9:16) has become the dominant format for social media and mobile-first experiences on platforms such as TikTok, Instagram Reels, and YouTube Shorts. This template deploys a single-pipeline live workflow that simultaneously produces **both a traditional horizontal (16:9) rendition and a vertical (9:16) rendition** from the same source, so a single MediaLive channel can serve both large-screen and mobile-portrait audiences.

The workflow is designed as a ready-to-use proof of concept (POC) and also wires up **AWS Elemental Inference** event capture (Smart Crop and Event Clipping) to CloudWatch Logs via EventBridge, making it a good starting point for AI-assisted vertical reframing experiments.

### What it deploys

- **AWS Elemental MediaLive** — a `SINGLE_PIPELINE` channel that loops an MP4 file from S3 and encodes two output groups:
  - **Horizontal (16:9)** ladder: 640x360, 960x540, 1280x720, 1920x1080 (H.264, CBR, 2s GOP) + AAC audio
  - **Vertical (9:16)** ladder: 540x960, 720x1280, 1080x1920 (H.264, CBR, 2s GOP) + AAC audio
- **AWS Elemental MediaPackage v2** — one Channel Group containing two CMAF channels (horizontal + vertical), each with its own origin endpoint and HLS manifest, plus origin endpoint policies restricting access to CloudFront.
- **Amazon CloudFront** — a distribution fronting MediaPackage v2 with Origin Access Control (OAC), a dedicated manifest cache policy and origin request policy (whitelisting `start`/`end` query strings for start-over playback).
- **EventBridge + CloudWatch Logs** — a rule capturing `aws.elemental-inference` events to a log group (7-day retention) for Smart Crop / Event Clipping observability.
- **IAM role** — the MediaLive service role granting access to S3, MediaPackage v2, Elemental Inference, and EventBridge.

### Architecture

```
                                        ┌─→ MediaPackage v2 (horizontal channel) ─┐
Input Source (S3) → MediaLive (SINGLE_PIPELINE) ─┤                                          ├─→ CloudFront → Player
   (MP4, looped)     ├─ 16:9 ladder            └─→ MediaPackage v2 (vertical channel) ───┘
                     └─ 9:16 ladder

Elemental Inference events → EventBridge → CloudWatch Logs
```

### Outputs

- `PlaybackUrlHorizontal` — HLS manifest URL for the 16:9 rendition
- `PlaybackUrlVertical` — HLS manifest URL for the 9:16 rendition (social / mobile)
- `CloudFrontDomain` — CloudFront distribution domain name
- `InferenceEventsLogGroup` — CloudWatch Log Group receiving Elemental Inference events

### Deployment

```bash
aws cloudformation create-stack \
  --stack-name my-vertical-workflow \
  --template-body file://CloudFormation_Single_EML-H264_Vertical_EMPv2_CloudFront.yaml \
  --parameters ParameterKey=InputS3Bucket,ParameterValue=my-input-bucket \
  --capabilities CAPABILITY_IAM \
  --region eu-west-1
```

> The template expects an input file named `formula_1-1.mp4` in the specified S3 bucket. Adjust the `Sources` URL in the template if your input file has a different name.

---

## Stopping workflows and cleaning up AWS resources after testing

To avoid unnecessary charges, make sure to stop the workflow when the MediaLive channel is not in use. Once your tests are complete, also ensure you delete the CloudFormation stack by following the steps below:

* Sign in to the AWS Console
* Go to CloudFormation
* Select the relevant stack
* Click “Delete”
* Confirm the action
