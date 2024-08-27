Certainly. I'll set up two microservices using Express.js and Mongoose for the Product and Order services, following a model-controller-route structure. I'll make the RPC functions reusable and modular, and structure the code for a professional, scalable application. Here's a comprehensive setup:

First, let's set up the directory structure:

```
ecommerce-microservices/
├── product-service/
│   ├── src/
│   │   ├── models/
│   │   ├── controllers/
│   │   ├── routes/
│   │   ├── config/
│   │   ├── utils/
│   │   └── app.js
│   ├── package.json
│   └── server.js
├── order-service/
│   ├── src/
│   │   ├── models/
│   │   ├── controllers/
│   │   ├── routes/
│   │   ├── config/
│   │   ├── utils/
│   │   └── app.js
│   ├── package.json
│   └── server.js
└── shared/
    └── rpc/
        └── rpcClient.js
```

Now, let's implement each part:

1. Shared RPC Client (shared/rpc/rpcClient.js):

```javascript
const amqp = require('amqplib');

class RPCClient {
  constructor(url) {
    this.url = url;
  }

  async initialize() {
    this.connection = await amqp.connect(this.url);
    this.channel = await this.connection.createChannel();
  }

  async call(queue, message) {
    const { channel } = this;
    const correlationId = this.generateUuid();
    const { queue: replyTo } = await channel.assertQueue('', { exclusive: true });

    return new Promise((resolve, reject) => {
      channel.consume(replyTo, (msg) => {
        if (msg.properties.correlationId === correlationId) {
          resolve(JSON.parse(msg.content.toString()));
        }
      }, { noAck: true });

      channel.sendToQueue(queue, Buffer.from(JSON.stringify(message)), {
        correlationId,
        replyTo,
      });
    });
  }

  generateUuid() {
    return Math.random().toString(36).substring(2, 15) + Math.random().toString(36).substring(2, 15);
  }

  async close() {
    await this.channel.close();
    await this.connection.close();
  }
}

module.exports = RPCClient;
```

2. Product Service:

a. Model (product-service/src/models/product.js):

```javascript
const mongoose = require('mongoose');

const productSchema = new mongoose.Schema({
  name: { type: String, required: true },
  description: { type: String, required: true },
  price: { type: Number, required: true },
  stock: { type: Number, required: true },
  category: { type: String, required: true },
}, { timestamps: true });

module.exports = mongoose.model('Product', productSchema);
```

b. Controller (product-service/src/controllers/productController.js):

```javascript
const Product = require('../models/product');

exports.createProduct = async (req, res) => {
  try {
    const product = new Product(req.body);
    await product.save();
    res.status(201).json(product);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getProduct = async (req, res) => {
  try {
    const product = await Product.findById(req.params.id);
    if (!product) return res.status(404).json({ error: 'Product not found' });
    res.json(product);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.updateProduct = async (req, res) => {
  try {
    const product = await Product.findByIdAndUpdate(req.params.id, req.body, { new: true });
    if (!product) return res.status(404).json({ error: 'Product not found' });
    res.json(product);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.deleteProduct = async (req, res) => {
  try {
    const product = await Product.findByIdAndDelete(req.params.id);
    if (!product) return res.status(404).json({ error: 'Product not found' });
    res.json({ message: 'Product deleted successfully' });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.listProducts = async (req, res) => {
  try {
    const products = await Product.find();
    res.json(products);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

c. Routes (product-service/src/routes/productRoutes.js):

```javascript
const express = require('express');
const productController = require('../controllers/productController');

const router = express.Router();

router.post('/', productController.createProduct);
router.get('/:id', productController.getProduct);
router.put('/:id', productController.updateProduct);
router.delete('/:id', productController.deleteProduct);
router.get('/', productController.listProducts);

module.exports = router;
```

d. RPC Server (product-service/src/utils/rpcServer.js):

```javascript
const amqp = require('amqplib');
const Product = require('../models/product');

class RPCServer {
  constructor(url) {
    this.url = url;
  }

  async start() {
    this.connection = await amqp.connect(this.url);
    this.channel = await this.connection.createChannel();

    const queue = 'product_queue';
    await this.channel.assertQueue(queue, { durable: false });

    console.log('RPC Server is waiting for requests');

    this.channel.consume(queue, async (msg) => {
      const request = JSON.parse(msg.content.toString());

      let response;
      switch (request.method) {
        case 'getProduct':
          response = await this.getProduct(request.params.id);
          break;
        case 'getProducts':
          response = await this.getProducts(request.params.ids);
          break;
        default:
          response = { error: 'Unknown method' };
      }

      this.channel.sendToQueue(msg.properties.replyTo, Buffer.from(JSON.stringify(response)), {
        correlationId: msg.properties.correlationId,
      });

      this.channel.ack(msg);
    });
  }

  async getProduct(id) {
    try {
      const product = await Product.findById(id);
      return product ? product.toObject() : { error: 'Product not found' };
    } catch (error) {
      return { error: error.message };
    }
  }

  async getProducts(ids) {
    try {
      const products = await Product.find({ _id: { $in: ids } });
      return products.map(product => product.toObject());
    } catch (error) {
      return { error: error.message };
    }
  }

  async close() {
    await this.channel.close();
    await this.connection.close();
  }
}

module.exports = RPCServer;
```

e. App (product-service/src/app.js):

```javascript
const express = require('express');
const mongoose = require('mongoose');
const productRoutes = require('./routes/productRoutes');
const RPCServer = require('./utils/rpcServer');
const config = require('./config');

const app = express();

mongoose.connect(config.mongoUri, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => console.error('Could not connect to MongoDB', err));

app.use(express.json());
app.use('/api/products', productRoutes);

const rpcServer = new RPCServer(config.rabbitmqUrl);
rpcServer.start();

module.exports = app;
```

f. Config (product-service/src/config/index.js):

```javascript
module.exports = {
  port: process.env.PORT || 3000,
  mongoUri: process.env.MONGO_URI || 'mongodb://localhost/product-service',
  rabbitmqUrl: process.env.RABBITMQ_URL || 'amqp://localhost',
};
```

g. Server (product-service/server.js):

```javascript
const app = require('./src/app');
const config = require('./src/config');

app.listen(config.port, () => {
  console.log(`Product Service running on port ${config.port}`);
});
```

3. Order Service:

a. Model (order-service/src/models/order.js):

```javascript
const mongoose = require('mongoose');

const orderSchema = new mongoose.Schema({
  userId: { type: String, required: true },
  products: [{
    productId: { type: String, required: true },
    quantity: { type: Number, required: true },
  }],
  totalAmount: { type: Number, required: true },
  status: { type: String, enum: ['pending', 'processing', 'shipped', 'delivered'], default: 'pending' },
}, { timestamps: true });

module.exports = mongoose.model('Order', orderSchema);
```

b. Controller (order-service/src/controllers/orderController.js):

```javascript
const Order = require('../models/order');
const RPCClient = require('../../../shared/rpc/rpcClient');
const config = require('../config');

const rpcClient = new RPCClient(config.rabbitmqUrl);

exports.createOrder = async (req, res) => {
  try {
    await rpcClient.initialize();
    const order = new Order(req.body);
    
    // Validate products and calculate total amount
    let totalAmount = 0;
    for (const item of order.products) {
      const product = await rpcClient.call('product_queue', { method: 'getProduct', params: { id: item.productId } });
      if (product.error) {
        throw new Error(`Invalid product: ${item.productId}`);
      }
      totalAmount += product.price * item.quantity;
    }
    
    order.totalAmount = totalAmount;
    await order.save();
    res.status(201).json(order);
  } catch (error) {
    res.status(400).json({ error: error.message });
  } finally {
    await rpcClient.close();
  }
};

exports.getOrder = async (req, res) => {
  try {
    const order = await Order.findById(req.params.id);
    if (!order) return res.status(404).json({ error: 'Order not found' });
    res.json(order);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};


exports.getOrderWithProducts = async (req, res) => {
  try {
    await rpcClient.initialize();

    const order = await Order.findById(req.params.id);
    if (!order) return res.status(404).json({ error: 'Order not found' });

    const productIds = order.products.map(item => item.productId);

    const batchMessage = {
      method: 'getProducts',
      params: { ids: productIds }
    };

    const products = await rpcClient.batchCall('product_queue', batchMessage);

    const productMap = new Map(products.map(p => [p._id.toString(), p]));

    const orderWithProducts = order.toObject();
    orderWithProducts.products = order.products.map(item => {
      const product = productMap.get(item.productId.toString());
      return {
        ...product,
        quantity: item.quantity,
        subtotal: product.price * item.quantity
      };
    });

    orderWithProducts.totalAmount = orderWithProducts.products.reduce((total, item) => total + item.subtotal, 0);

    res.json(orderWithProducts);
  } catch (error) {
    res.status(400).json({ error: error.message });
  } finally {
    await rpcClient.close();
  }
};

exports.updateOrderStatus = async (req, res) => {
  try {
    const order = await Order.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status },
      { new: true }
    );
    if (!order) return res.status(404).json({ error: 'Order not found' });
    res.json(order);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.listOrders = async (req, res) => {
  try {
    const orders = await Order.find();
    res.json(orders);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

c. Routes (order-service/src/routes/orderRoutes.js):

```javascript
const express = require('express');
const orderController = require('../controllers/orderController');

const router = express.Router();

router.post('/', orderController.createOrder);
router.get('/:id', orderController.getOrder);
router.get('/:id/with-products', orderController.getOrderWithProducts); 
router.put('/:id/status', orderController.updateOrderStatus);
router.get('/', orderController.listOrders);

module.exports = router;
```

d. App (order-service/src/app.js):

```javascript
const express = require('express');
const mongoose = require('mongoose');
const orderRoutes = require('./routes/orderRoutes');
const config = require('./config');

const app = express();

mongoose.connect(config.mongoUri, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => console.error('Could not connect to MongoDB', err));

app.use(express.json());
app.use('/api/orders', orderRoutes);

module.exports = app;
```

e. Config (order-service/src/config/index.js):

```javascript
module.exports = {
  port: process.env.PORT || 3001,
  mongoUri: process.env.MONGO_URI || 'mongodb://localhost/order-service',
  rabbitmqUrl: process.env.RABBITMQ_URL || 'amqp://localhost',
};
```

f. Server (order-service/server.js):

```javascript
const app = require('./src/app');
const config = require('./src/config');

app.listen(config.port, () => {
  console.log(`Order Service running on port ${config.port}`);
});
```

This setup provides a scalable and modular structure for both the Product and Order services. The RPC mechanism is abstracted into reusable classes (RPCClient and RPCServer) that can be easily extended for other services.

Key points:

1. The code follows the MVC pattern with separate models, controllers, and routes.
2. The RPC mechanism is implemented in a way that allows easy extension for other methods.
3. Configuration is centralized and environment-variable based for easy deployment.
4. Error handling is implemented throughout the services.
5. The structure allows for easy addition of new features or microservices.

To run these services, you'll need to:

1. Set up MongoDB databases for each service.
2. Set up a RabbitMQ server.
3. Install dependencies (express, mongoose, amqplib) for each service.
4. Run each service separately.

This structure provides a solid foundation for a serious, scalable e-commerce backend. You can easily extend it by adding more services, implementing authentication, and adding more complex business logic as needed.