# volvocars.com leaks license plate information

## TL;DR

While looking up vehicle model information on volvocars.com using a [VIN](https://en.wikipedia.org/wiki/Vehicle_identification_number) (Vehicle Identification Number), the server returns the registration number (aka. license plate) of the given vehicle as well (if Volvo's database has it, which many times it does since dealers tend to record the reg. number for newly sold vehicles).

## The details

The Volvo Cars website features a "Car information" page.

E.g. in English: [https://www.volvocars.com/uk/support/topic/f2d15ef7057c40c8a570b75a2a091698/](https://www.volvocars.com/uk/support/topic/f2d15ef7057c40c8a570b75a2a091698/)

This allows a visitor to look up vehicle model information based on a VIN or a registration number (aka. license plate number).

It uses a Volvo Cars GraphQL endpoint to fetch the necessary information: `https://www.volvocars.com/api/support/graphql`

When the user enters a VIN on the website and clicks the "Find your car" button, the JavaScript code sends the following request (I've kept the minimal working set of HTTP headers for simplicity):

```bash
curl "https://www.volvocars.com/api/support/graphql" \
  -X POST \
  -H 'Content-Type: application/json' \
  --data-raw $'{"query":"\\n  \\n  fragment carRelease on CarRelease {\\n    id\\n    displayName\\n    modelSlug\\n    modelCode\\n    structureWeek\\n    modelYear\\n    isLatestRelease\\n    projectCode\\n    registrationNumber\\n  }\\n\\n  query CarDetails($vin: String\041) {\\n    carByVin(vin: $vin) {\\n      ...carRelease\\n    }\\n  }\\n","variables":{"vin":"YV1.............."},"operationName":"CarDetails"}'
```

The response looks like this:

```json
{
  "data": {
    "carByVin": {
      "id": "gr-V316-18w17",
      "displayName": "XC40",
      "modelSlug": "xc40",
      "modelCode": "536",
      "structureWeek": "18w17",
      "modelYear": 2019,
      "isLatestRelease": false,
      "projectCode": "V316",
      "registrationNumber": "GEF9587"
    }
  }
}
```

I see a problem with this: the query can request the "registrationNumber" field even if the lookup is done through a VIN.

Although:

- Both the registration number (aka. license plates) and the VIN are publicly visible on vehicles (the VIN is usually visible in one of the bottom corners of the windshield).
- Both the structure of VIN numbers and the structure of registration numbers is public information.

However:

- The mapping between VINs and registration numbers is not public information!
- It is trivial to generate all VINs for all Volvo Cars models that were ever manufactured.
- Using this API somebody could query registration numbers for all VolvoCars VINs.

The GraphQL API doesn't contain the VIN in the response, so the registrationNumber->VIN mapping via this API is (luckily) not possible.

I suspect that this information leak was not intentional. Also it might conflict with a couple of countries' laws/regulations.

I've reported this to `responsible-disclosure@volvocars.com` on July 18 2026 (90+ days ago), but they didn't care to respond. I've reported it to `hu.info@volvocars.com` a month later (17th Aug 2026). They did respond on 26th Aug. (I've translated it from Hungarian to English):

> We would like to inform you that we have forwarded the reported issue to our relevant team of experts, and our IT department is currently investigating the matter. Should further information be required during the investigation, our colleagues will, of course, contact you.

When I asked whether they'll contact me with the result of their investigation/evaluation even if they don't have further questions, I got no reply. I made it clear to them that I'd publish my findings, so probably they didn't consider this to be an issue. And here we are.

I've found that the same API is available also at the `https://support-content.volvo.care/api/graphql` endpoint and is used by the vehicles themselves. Or at least the two endpoints share the same GraphQL schema and respond the same way to the same queries. So my findings regarding the `https://www.volvocars.com/api/support/graphql` endpoint also concern the `https://support-content.volvo.care/api/graphql` endpoint.

The difference (as seen from outside):

- The `https://www.volvocars.com/api/support/graphql` endpoint is protected by the Akamai CDN (e.g. access from the Tor network is forbidden).
- The `https://support-content.volvo.care/api/graphql` endpoint seems to be not protected by a professional 3rd party service (can be called from Tor as well), the requests seem to directly hit the Istio-Envoy proxy layer. The response lacks the `akamai-request-bc` header (among other things) and has a `server` header.

The response headers of the former are:

```plain
HTTP/2 200 
content-type: application/json
vcc-api-operationid: de260382-b395-4aa8-b1e6-eb8a4e4821ce
access-control-allow-origin: *
access-control-allow-methods: GET, PUT, POST, DELETE, OPTIONS
access-control-allow-headers: *
x-envoy-upstream-service-time: 454
cache-control: max-age=3600
expires: Mon, 21 Oct 2026 16:15:34 GMT
date: Mon, 21 Oct 2026 15:15:34 GMT
content-length: 219
alt-svc: h3=":443"; ma=93600
akamai-request-bc: [a=104.83.4.189,b=487372057,c=g,n=AT__VIENNA,o=95923]
x-content-type-options: nosniff
strict-transport-security: max-age=15768000 ; includeSubDomains ; preload
```

The headers for the latter are:

```plain
HTTP/2 200 
date: Mon, 21 Oct 2026 15:15:34 GMT
vary: Accept-Encoding
vcc-api-operationid: de260382-b395-4aa8-b1e6-eb8a4e4821ce
content-type: application/json
access-control-allow-origin: *
access-control-allow-methods: GET, PUT, POST, DELETE, OPTIONS
access-control-allow-headers: *
content-length: 219
x-envoy-upstream-service-time: 8
server: istio-envoy
```

There's another query with a VIN input and returns the registrationNumber as well.

```bash
curl "https://support-content.volvo.care/api/graphql" \
  -X POST \
  -H "Content-Type: application/json" \
  --data-raw $'{"query":"\\n  \\n  fragment softwareUpdateInformation on SoftwareUpdateInformation {\\n    carModelYear\\n    id\\n    message\\n    structureWeek\\n    vehicleCode\\n    vehicleName\\n    registrationNumber\\n  }\\n\\n  query CarDetails($vin: String\041) {\\n    carNetworkStatusByVin(vin: $vin) {\\n      ...softwareUpdateInformation\\n    }\\n  }\\n","variables":{"vin":"YV1.............."},"operationName":"CarDetails"}'
```

Returns:

```json
{
  "data": {
    "carNetworkStatusByVin": {
      "carModelYear": "2019",
      "id": "44878b6a-5915-4afd-b457-532d4bc55b04",
      "message": "SW updated - Not affected",
      "structureWeek": "18w33",
      "vehicleCode": "536",
      "vehicleName": "XC40",
      "registrationNumber": "GEF9587"
    }
  }
}
```
