Part A – Risk-based API test design

Endpoint and method: POST /orders Why it is critical:
It creates a real order, which is a business-critical write operation.
It drives downstream processes such as fulfilment, payment validation, and reporting.
It has validation rules and a financial calculation (total) that must be correct.
If it fails, the customer cannot complete the purchase.
Preconditions:

Valid bearer token is supplied.
Customer ID is a valid UUID.
At least one order item is provided.
Item SKU is valid/known to the system.
Quantity is at least 1.
Unit price is zero or greater.


Part B – Execute or demonstrate one scenario

Chosen scenario: PATCH /orders/{orderId} – valid status transition

Automation path Scope:

Create an order.
Validate order is in initial status.
Update status to a valid next state.
Assert response and persisted result.
Negative validation: illegal transition returns 409.

# inspiredtesting


test('PATCH /orders/{orderId} updates status for a valid transition', async () => {
  const token = process.env.API_TOKEN;
  const customerId = crypto.randomUUID();

  const createOrderPayload = {
    customerId,
    items: [
      { sku: 'SKU-123', quantity: 2, unitPrice: 12.5 }
    ]
  };

  const createRes = await fetch('https://api.assessment.example.com/v1/orders', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(createOrderPayload)
  });

  expect(createRes.status).toBe(201);
  const createdOrder = await createRes.json();
  expect(createdOrder.status).toBe('pending');

  const orderId = createdOrder.id;

  const updateRes = await fetch(`https://api.assessment.example.com/v1/orders/${orderId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: 'paid' })
  });

  expect(updateRes.status).toBe(200);

  const updatedOrder = await updateRes.json();
  expect(updatedOrder.id).toBe(orderId);
  expect(updatedOrder.status).toBe('paid');
});

test('PATCH /orders/{orderId} rejects an illegal transition', async () => {
  const token = process.env.API_TOKEN;
  const customerId = crypto.randomUUID();

  const createOrderPayload = {
    customerId,
    items: [
      { sku: 'SKU-456', quantity: 1, unitPrice: 25.00 }
    ]
  };

  const createRes = await fetch('https://api.assessment.example.com/v1/orders', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(createOrderPayload)
  });

  const createdOrder = await createRes.json();
  const orderId = createdOrder.id;

  const invalidTransitionRes = await fetch(`https://api.assessment.example.com/v1/orders/${orderId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: 'shipped' }) // illegal if current is pending
  });

  expect(invalidTransitionRes.status).toBe(409);
  const errorBody = await invalidTransitionRes.json();
  expect(errorBody).toBeDefined();
});



Part C – Contract awareness

One API change that would break a consumer is changing the Order.total field from a numeric value to a string, or changing required fields in the response contract without versioning. A consumer using the total value for calculations, for example, would fail if it receives a string and tries to perform arithmetic, or it might break UI logic that expects a number. A consumer contract test would detect this by validating the schema against the agreed OpenAPI contract, asserting field types and required properties for both successful and error responses, and failing if the producer introduces a breaking change such as a type change, a missing required field, or a different enum value.
