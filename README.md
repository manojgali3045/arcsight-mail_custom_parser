ArcSight Custom Mail Parser

A custom ArcSight FlexConnector parser for syslog events from a mail platform built with Postfix, Cyrus IMAP, Amavisd-new, OpenDKIM, OpenDMARC and related services.

The parser uses a shared base pattern and service-specific submessage patterns to classify mail activity and map supported fields into the ArcSight event schema.

## Repository contents

| File | Purpose |
|---|---|
| `omail_v4.sdkrfilereader.properties` | Custom mail-platform syslog parser; the reviewed file identifies itself as v4.3.0 |
| `README.md` | Scope, validation results, limitations and deployment guidance |

## Supported event families

- **Postfix:** SMTP authentication failures, authenticated submissions, client connections, queue activity, message IDs, delivery results, rejections and TLS events.
- **Cyrus:** mailbox authentication, client identification, TLS sessions, mailbox delivery, expunge activity and DAV requests.
- **Amavis:** scan verdicts, spam-tag results and DKIM-related messages.
- **Mail authentication and policy:** OpenDKIM, OpenDMARC, policyd-spf and postfwd2.
- **Supporting services:** saslauthd, UFW kernel messages, SSH authentication, ClamAV and FreshClam.

Support means patterns are present; it does not mean every event path has been exercised in a live connector.

## Parser design

1. Extract the process family, process suffix, PID and message body.
2. Select the corresponding service submessage block.
3. Apply specific event patterns before fallback patterns.
4. Map extracted values and event classifications into ArcSight fields.

Common mappings include:

| Field | Intended use |
|---|---|
| `deviceVendor` | `OMAIL` |
| `deviceProduct` | Service or application family |
| `deviceProcessName` | Process name, including service suffix |
| `deviceCustomString1` | Queue ID where available |
| `deviceCustomString2` | Message ID where available |
| `deviceCustomString3` | Result or result code, depending on event type |
| `deviceCustomString4` | Event-specific detail such as relay, mailbox or TLS information |
| `deviceCustomString5` | Event-specific session, task or other detail |
| `deviceCustomString6` | Authentication mechanism where available |

Use each event's custom-field label when interpreting overloaded fields. Cyrus login accounts are mapped to `destinationUserName` in the reviewed version.

## Validation performed

An offline review examined a capture containing **4,781,723 syslog records**. Its first and last source timestamps were approximately **9 hours 55 minutes apart**.

An independent Python replay of the reviewed v4.3 regular expressions found **zero base-pattern failures**. A separate Logger export confirmed OMAIL event classification for 8,928 exported records.

These results have important limits:

- A matching base pattern does not prove correct field extraction or event meaning.
- Python regex replay does not validate ArcSight runtime conversions or mapping functions.
- The Logger export omitted several important fields, preventing complete field-level verification.
- Catch-all patterns contribute heavily to the matching total.
- No blocked-malware event was observed in the reviewed long capture, so that path remains unverified by this dataset.

Production captures and account details are not needed to use this repository and should not be submitted with issues or contributions.

## Known limitations

### Fallback classification and filtering

The reviewed parser labels unmatched Cyrus and Amavis messages as debug. Those buckets represented approximately **88.06%** of the reviewed capture. They can include unfamiliar errors as well as ordinary trace messages.

Do not assume that dropping entire debug categories is lossless. A planned improvement is to retain unknown messages as `other` and classify only explicitly recognized noise as debug.

### Event semantics

The reviewed version needs further refinement for:

- HTTP outcomes: the Cyrus HTTP pattern currently assigns success independently of the captured status code.
- SMTP actions: the NOQUEUE pattern groups reject, discard, hold and warn under a rejection classification.
- TLS direction: inbound and outbound peer addresses require direction-aware interpretation.

### Correlation and attribution

- Scope Queue IDs by host and time; they are not global identifiers.
- Treat Message IDs as correlation hints rather than guaranteed unique or trustworthy identifiers.
- Webmail-mediated connections may expose the webmail server's IP rather than the end user's IP.
- Authentication companion messages can represent the same attempt; avoid counting each line as a separate login failure.
- A Sieve login does not establish that a mailbox forwarding rule changed.

## Deployment guidance

This is a custom parser and must be validated against the intended connector version and local event formats.

1. Preserve the existing parser and connector configuration for rollback.
2. Review the parser for environment-specific comments and assumptions.
3. Test representative sanitized events in an isolated connector environment.
4. Verify both classification and extracted fields in the connector and destination.
5. Check timestamps, addresses, account fields, identifiers, outcomes and severity.
6. Review unmatched events and filtering behaviour before production rollout.
7. Validate standby behaviour and rollback using the deployment's own procedures.

Use the FlexConnector documentation matching your installed version for parser placement, loading and testing. This repository does not include the active connector configuration or establish a universal installation procedure.

## Roadmap

- Separate unknown events from recognized debug messages.
- Add synthetic log samples with expected normalized fields.
- Add regression tests for optional fields, IPv6, truncated messages and format variations.
- Correct event outcome and direction mappings.
- Publish version-specific runtime validation results.

## Contributing

For an unmatched or incorrectly mapped event, provide:

- A synthetic or carefully sanitized example preserving the message structure.
- The expected event type and fields.
- The parser and connector versions.
- The observed result.

Remove production usernames, email addresses, hostnames, internal addresses, identifiers and credentials before sharing examples. Do not attach raw production captures.

## References

- [ArcSight FlexConnector: Creating a Parser](https://www.microfocus.com/documentation/arcsight/arcsight-smartconnectors-8.4/flexconn_devguide/Content/convertFlex/Chapter_FlexConnector_Parsers.htm)
- [ArcSight FlexConnector: Using Sub-Messages](https://www.microfocus.com/documentation/arcsight/arcsight-smartconnectors-8.4/flexconn_devguide/Content/convertFlex/Sub_Messages.htm)
