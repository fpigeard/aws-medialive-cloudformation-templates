# AWS MediaLive CloudFormation Templates

Ready-to-use AWS CloudFormation templates for live streaming workflows using AWS Elemental MediaLive, AWS Elemental MediaPackage v2, and Amazon CloudFront.

## Templates

| Template | Codec | Rate Control | Latency |
|---|---|---|---|
| [Single H264](CloudFormation_Single_EML-H264_EMPv2_CloudFront.yaml) | H.264 | CBR | Single-pipeline |
| [Dual H264 CBR](CloudFormation_Dual-CBR_EML-H264_EMPv2_CloudFront.yaml) | H.264 | CBR | Standard |
| [Dual H264 QVBR](CloudFormation_Dual-QVBR_EML-H264_EMPv2_CloudFront.yaml) | H.264 | QVBR | Standard |
| [Dual H264 QVBR LL-HLS](CloudFormation_Dual-QVBR_LL-HLS_EML-H264_EMPv2_CloudFront.yaml) | H.264 | QVBR | Low-Latency |
| [HEVC](CloudFormation_EML-HEVC_MediaPackagev2_CloudFront.yaml) | H.265 (HEVC) | CBR | Single-pipeline |

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
