# Architectural decision records

Version 0.2.2 is a lightweight extension to the original web based zotero tools.
This is a bit fragile and

## Local first, Zotero 10+

From versions 0.2.2 the main requirement is a locally running Zotero with v10+
with "Allow other applications on this computer to communicate with Zotero" ticked in settings > advanced.

This results in a localhost:21339 port - it is not configurable.

If this is not reachable then we fail fast with enough info to help an agent guide the user through the process of installing zoterp 10+, enabling the API and making sure the port is not blocked.

If the API is there but zotero is a version <10 we fail fast with instructions to upgrade zotero.

## Web API usage only if explicitly configured

Using web is only available in the precise situation that a user requests it, i.e. a ZOTERO_WEB_API_KEY env variable must be set, and ZOTERO_WEB is true.

No locally held api key in configuration. No zotero provider.

## Fixing metadata and PDF urls.



https://help.openalex.org/api/selecting-fields/


https://api.openalex.org/works/pmid:36530491?select=title,ids,primary_location,type,locations


works can be doi: or pmid: (pmcid does not usually work)

PMID: 35761225
