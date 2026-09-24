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
