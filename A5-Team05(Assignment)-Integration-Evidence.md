# Assignment 5: Interface between Groups

## Group Integration Evidence

| Item | Details |
| --- | --- |
| Team Name | Team 05 |
| Partner Team | Team 06 |
| Document Name | `A5-Team05-Integration-Evidence` |

## 1. Consumer Proof

Evidence from the Consumer side for calling the Partner Team API:

- Partner URL, request timestamp, and response verification
- Raw submission JSON output retrieved from the Partner
- Request test using a placeholder/invalid integration key (`test-key`)

![Request using a placeholder integration key with a 401 Unauthorized response](images/images2.png)

- Fetching submissions from the Partner URL (`https://prd-team06...`) using a valid key

![Request using a valid integration key and the returned submissions](images/images3.png)

- Request test using an invalid integration key

![Request using an invalid integration key with a 401 Unauthorized response](images/images4.png)

- Request test using a non-existent assignment ID

![Request using a non-existent assignment ID with a 404 Not Found response](images/images5.png)

Raw submission JSON output retrieved from the Partner:

![Raw submission JSON output retrieved from the Partner](images/images1.png)

## 2. Provider Proof

Evidence from the Provider side for the Team 05 endpoint:

- Endpoint URL, internal request logs, and partner confirmation
- Vercel request logs showing incoming requests to our API

![Vercel request logs for the API](images/images6.png)

- Postman request to our endpoint (`/api/team06/assignments/...`) returning data from the Partner

![Postman request to the Team 05 endpoint returning data from the Partner](images/images7.png)

## 3. Webhook Receiver

Evidence for receiving the Webhook:

- Incoming payload sent to `/api/webhooks/receive`

![Incoming Webhook payload and response from the receiver](images/images8.png)

- Webhook secret verification
- Vercel server logs recording the incoming Webhook execution

![Vercel server logs for the Webhook receiver](images/images9.png)

## 4. Webhook Sender

Evidence for sending the Webhook:

- Internal trigger action
- Outgoing payload
- Partner response verification
- Webhook payload sent to the Partner endpoint (`https://prd-team06.../api/webhooks/receive`) with a secret header

![Webhook sent to the Partner with a secret header and response](images/images10.png)

![Response from the Team 05 Webhook endpoint](images/images11.png)

## 5. Idempotency Proof

Comparison of Request 1 and Request 2 to verify single creation and deduplication:

- **First request:** Initial Webhook payload processed with `idempotent: false`
- **Second request:** Duplicate payload sent with `idempotent: true`

![First request result: idempotent false](images/images8.png)

![Second request result: idempotent true](images/images12.png)

## 6. Degradation Proof

Evidence of handling Partner service failure:

- The system returns a fallback JSON response when the Partner service is unavailable
- The result must include `fallback: true`

![Fallback JSON response when the Partner service is unavailable](images/images13.png)
