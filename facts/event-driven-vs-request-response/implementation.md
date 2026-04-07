# Code Examples: Kafka & RabbitMQ

## Kafka: Command + Event Pattern

**Kafka setup for order processing:**
```python
from kafka import KafkaProducer, KafkaConsumer
import json
import uuid

# Producer: Create order command
def place_order_command(user_id, items):
    producer = KafkaProducer(
        bootstrap_servers=['localhost:9092'],
        value_serializer=lambda v: json.dumps(v).encode('utf-8')
    )

    # Command: synchronous validation
    order_id = str(uuid.uuid4())
    order = {
        'order_id': order_id,
        'user_id': user_id,
        'items': items,
        'total': sum(item['price'] for item in items),
        'status': 'pending'
    }

    # Save to DB (synchronous)
    db.orders.insert_one(order)

    # Send async events
    producer.send('orders.events', value={
        'event_type': 'OrderPlaced',
        'order_id': order_id,
        'user_id': user_id,
        'timestamp': datetime.now().isoformat()
    })

    # Return immediately to user
    return {'order_id': order_id, 'status': 'confirmed'}

# Consumer: PaymentService
def start_payment_consumer():
    consumer = KafkaConsumer(
        'orders.events',
        bootstrap_servers=['localhost:9092'],
        value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        group_id='payment-service'
    )

    for message in consumer:
        event = message.value

        if event['event_type'] == 'OrderPlaced':
            order_id = event['order_id']
            order = db.orders.find_one({'_id': order_id})

            try:
                # Process payment asynchronously
                transaction = charge_card(order['user_id'], order['total'])

                # Publish success event
                producer.send('orders.events', value={
                    'event_type': 'PaymentCharged',
                    'order_id': order_id,
                    'transaction_id': transaction['id'],
                    'amount': order['total']
                })

                # Update order
                db.orders.update_one(
                    {'_id': order_id},
                    {'$set': {'status': 'paid'}}
                )

            except Exception as e:
                producer.send('orders.events', value={
                    'event_type': 'PaymentFailed',
                    'order_id': order_id,
                    'error': str(e)
                })
                db.orders.update_one(
                    {'_id': order_id},
                    {'$set': {'status': 'failed'}}
                )

# Consumer: InventoryService
def start_inventory_consumer():
    consumer = KafkaConsumer(
        'orders.events',
        bootstrap_servers=['localhost:9092'],
        value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        group_id='inventory-service'
    )

    for message in consumer:
        event = message.value

        if event['event_type'] == 'PaymentCharged':
            order_id = event['order_id']
            order = db.orders.find_one({'_id': order_id})

            try:
                # Reserve inventory
                for item in order['items']:
                    db.inventory.update_one(
                        {'sku': item['sku']},
                        {'$inc': {'reserved': item['quantity']}}
                    )

                producer.send('orders.events', value={
                    'event_type': 'InventoryReserved',
                    'order_id': order_id
                })

            except Exception as e:
                # Compensation: request refund
                producer.send('orders.events', value={
                    'event_type': 'RefundRequested',
                    'order_id': order_id
                })
```

## RabbitMQ: Reply Queue Pattern (Request-Reply RPC)

```python
import pika
import json
import uuid

class RPCClient:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()

        # Create reply queue for responses
        result = self.channel.queue_declare(
            queue='',
            exclusive=True
        )
        self.callback_queue = result.method.queue

        # Listen for responses
        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True
        )

        self.response = None
        self.corr_id = None

    def on_response(self, ch, method, props, body):
        if props.correlation_id == self.corr_id:
            self.response = json.loads(body)

    def call(self, command):
        self.response = None
        self.corr_id = str(uuid.uuid4())

        # Send RPC command
        self.channel.basic_publish(
            exchange='',
            routing_key='payment.rpc',
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=self.corr_id,
            ),
            body=json.dumps(command)
        )

        # Wait for response (blocking, but asynchronous underneath)
        while self.response is None:
            self.connection.process_data_callbacks()

        return self.response


class RPCServer:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='payment.rpc')
        self.channel.basic_consume(
            queue='payment.rpc',
            on_message_callback=self.on_request
        )

    def on_request(self, ch, method, props, body):
        command = json.loads(body)

        try:
            # Process RPC request
            if command['action'] == 'charge':
                result = charge_card(
                    command['card_token'],
                    command['amount']
                )
                response = {
                    'success': True,
                    'transaction_id': result['id']
                }
        except Exception as e:
            response = {'success': False, 'error': str(e)}

        # Send response back to reply_to queue
        self.channel.basic_publish(
            exchange='',
            routing_key=props.reply_to,
            properties=pika.BasicProperties(
                correlation_id=props.correlation_id
            ),
            body=json.dumps(response)
        )

        ch.basic_ack(delivery_tag=method.delivery_tag)

    def start(self):
        print('[*] Waiting for RPC requests...')
        self.channel.start_consuming()


# Usage
if __name__ == '__main__':
    # Client
    rpc_client = RPCClient()
    response = rpc_client.call({
        'action': 'charge',
        'card_token': 'tok_4242',
        'amount': 9999
    })
    print(f"Payment response: {response}")

    # Server (run in another process)
    # rpc_server = RPCServer()
    # rpc_server.start()
```

## Choreography (Saga) with Kafka

```python
# Define events
class OrderPlaced:
    def __init__(self, order_id, user_id, total):
        self.order_id = order_id
        self.user_id = user_id
        self.total = total

class PaymentCharged:
    def __init__(self, order_id, transaction_id):
        self.order_id = order_id
        self.transaction_id = transaction_id

class PaymentFailed:
    def __init__(self, order_id, error):
        self.order_id = order_id
        self.error = error

# Saga orchestrator
class OrderSaga:
    def __init__(self, producer):
        self.producer = producer
        self.sagas = {}  # order_id -> saga state

    def start_order(self, order_id, user_id, total):
        self.sagas[order_id] = {
            'status': 'pending',
            'payment_attempted': False,
            'inventory_reserved': False
        }

        self.producer.send('orders.saga', value={
            'saga_id': order_id,
            'step': 'REQUEST_PAYMENT',
            'payload': {
                'user_id': user_id,
                'amount': total
            }
        })

    def on_payment_charged(self, order_id, transaction_id):
        if order_id not in self.sagas:
            return

        saga = self.sagas[order_id]
        saga['payment_attempted'] = True

        # Next step: reserve inventory
        self.producer.send('orders.saga', value={
            'saga_id': order_id,
            'step': 'RESERVE_INVENTORY',
            'payload': {'order_id': order_id}
        })

    def on_payment_failed(self, order_id, error):
        if order_id not in self.sagas:
            return

        saga = self.sagas[order_id]
        saga['status'] = 'failed'

        # Compensation: don't do anything (no payment to refund)
        self.producer.send('orders.saga', value={
            'saga_id': order_id,
            'step': 'ORDER_FAILED',
            'reason': error
        })

    def on_inventory_reserved(self, order_id):
        saga = self.sagas[order_id]
        saga['inventory_reserved'] = True
        saga['status'] = 'confirmed'

    def on_inventory_failed(self, order_id):
        saga = self.sagas[order_id]
        saga['status'] = 'compensation_required'

        # Compensation: refund payment
        self.producer.send('orders.saga', value={
            'saga_id': order_id,
            'step': 'REQUEST_REFUND',
            'reason': 'inventory_unavailable'
        })
```

## HTTP Endpoint returning immediately

```python
from fastapi import FastAPI
from kafka import KafkaProducer

app = FastAPI()
producer = KafkaProducer(bootstrap_servers=['localhost:9092'])

@app.post("/orders")
def create_order(request: OrderRequest):
    # 1. Validate (synchronous, fast)
    if not validate_items(request.items):
        return {"error": "Invalid items"}, 400

    # 2. Create order in DB
    order = Order(user_id=request.user_id, items=request.items)
    db.session.add(order)
    db.session.commit()

    # 3. Return immediately
    response = {
        "order_id": str(order.id),
        "status": "confirmed",
        "total": order.total
    }

    # 4. Publish async events (fire and forget)
    producer.send_and_forget('orders.events', {
        'event_type': 'OrderPlaced',
        'order_id': str(order.id)
    })

    return response, 200
```

This approach ensures users get fast responses while background services process asynchronously.
