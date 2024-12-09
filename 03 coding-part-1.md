# application based on the technical documentation. Let's start with the core components.



```javascript
// server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

const app = express();

app.use(cors());
app.use(express.json());

// Database Models
const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  addresses: [{
    street: String,
    city: String,
    state: String,
    zipCode: String
  }],
  orders: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Order' }]
});

const productSchema = new mongoose.Schema({
  name: String,
  description: String,
  price: Number,
  stock: Number,
  images: [String],
  variants: [{
    name: String,
    price: Number,
    stock: Number
  }]
});

const orderSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  items: [{
    product: { type: mongoose.Schema.Types.ObjectId, ref: 'Product' },
    quantity: Number,
    price: Number
  }],
  total: Number,
  status: String,
  shippingAddress: {
    street: String,
    city: String,
    state: String,
    zipCode: String
  },
  paymentIntent: String
});

const User = mongoose.model('User', userSchema);
const Product = mongoose.model('Product', productSchema);
const Order = mongoose.model('Order', orderSchema);

// API Routes
app.post('/api/auth/register', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = new User({ email, password: await bcrypt.hash(password, 10) });
    await user.save();
    const token = jwt.sign({ userId: user._id }, process.env.JWT_SECRET);
    res.json({ token });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

app.post('/api/orders', async (req, res) => {
  try {
    const { items, shippingAddress } = req.body;
    const total = await calculateOrderTotal(items);
    
    const paymentIntent = await stripe.paymentIntents.create({
      amount: total * 100,
      currency: 'usd'
    });

    const order = new Order({
      user: req.user._id,
      items,
      total,
      shippingAddress,
      paymentIntent: paymentIntent.id,
      status: 'pending'
    });

    await order.save();
    res.json({ clientSecret: paymentIntent.client_secret });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Start server
mongoose.connect(process.env.MONGODB_URI);
app.listen(3000, () => console.log('Server running on port 3000'));

```



```tsx
import React, { useState } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Elements } from '@stripe/stripe-js';
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(process.env.REACT_APP_STRIPE_PUBLIC_KEY);

const App = () => {
  const [cart, setCart] = useState([]);

  return (
    <BrowserRouter>
      <div className="min-h-screen bg-gray-50">
        <nav className="bg-white shadow-sm">
          <div className="max-w-7xl mx-auto px-4 py-4 flex justify-between items-center">
            <h1 className="text-xl font-bold">Water Purification Systems</h1>
            <div className="flex items-center space-x-4">
              <CartIcon items={cart.length} />
              <UserMenu />
            </div>
          </div>
        </nav>

        <main className="max-w-7xl mx-auto px-4 py-8">
          <Routes>
            <Route path="/" element={<Home />} />
            <Route path="/products" element={<ProductList />} />
            <Route path="/cart" element={<Cart cart={cart} setCart={setCart} />} />
            <Route
              path="/checkout"
              element={
                <Elements stripe={stripePromise}>
                  <Checkout cart={cart} />
                </Elements>
              }
            />
          </Routes>
        </main>
      </div>
    </BrowserRouter>
  );
};

const ProductList = () => {
  const [products, setProducts] = useState([]);

  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
      {products.map(product => (
        <div key={product._id} className="bg-white rounded-lg shadow p-6">
          <img
            src={product.images[0]}
            alt={product.name}
            className="w-full h-48 object-cover rounded"
          />
          <h2 className="mt-4 text-lg font-semibold">{product.name}</h2>
          <p className="mt-2 text-gray-600">{product.description}</p>
          <div className="mt-4 flex justify-between items-center">
            <span className="text-xl font-bold">${product.price}</span>
            <button className="bg-blue-600 text-white px-4 py-2 rounded">
              Add to Cart
            </button>
          </div>
        </div>
      ))}
    </div>
  );
};

const Checkout = ({ cart }) => {
  const stripe = useStripe();
  const elements = useElements();

  const handleSubmit = async (event) => {
    event.preventDefault();
    const { error, paymentMethod } = await stripe.createPaymentMethod({
      type: 'card',
      card: elements.getElement('card')
    });

    if (!error) {
      // Handle successful payment
    }
  };

  return (
    <div className="max-w-lg mx-auto">
      <form onSubmit={handleSubmit} className="space-y-6">
        <div className="bg-white rounded-lg shadow p-6">
          <h2 className="text-xl font-semibold mb-4">Payment Information</h2>
          <CardElement className="p-4 border rounded" />
        </div>
        <button
          type="submit"
          className="w-full bg-blue-600 text-white py-3 rounded-lg"
        >
          Pay Now
        </button>
      </form>
    </div>
  );
};

export default App;

```



```javascript
// App.js
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { Provider as PaperProvider } from 'react-native-paper';
import { StripeProvider } from '@stripe/stripe-react-native';

const Stack = createNativeStackNavigator();

const App = () => {
  return (
    <StripeProvider publishableKey={process.env.STRIPE_PUBLIC_KEY}>
      <PaperProvider>
        <NavigationContainer>
          <Stack.Navigator>
            <Stack.Screen name="Home" component={HomeScreen} />
            <Stack.Screen name="Products" component={ProductScreen} />
            <Stack.Screen name="Cart" component={CartScreen} />
            <Stack.Screen name="Checkout" component={CheckoutScreen} />
          </Stack.Navigator>
        </NavigationContainer>
      </PaperProvider>
    </StripeProvider>
  );
};

// screens/ProductScreen.js
import { View, FlatList, StyleSheet } from 'react-native';
import { Card, Title, Paragraph, Button } from 'react-native-paper';

const ProductScreen = ({ navigation }) => {
  const [products, setProducts] = useState([]);

  const renderProduct = ({ item }) => (
    <Card style={styles.card}>
      <Card.Cover source={{ uri: item.images[0] }} />
      <Card.Content>
        <Title>{item.name}</Title>
        <Paragraph>{item.description}</Paragraph>
        <Title>${item.price}</Title>
      </Card.Content>
      <Card.Actions>
        <Button onPress={() => addToCart(item)}>Add to Cart</Button>
      </Card.Actions>
    </Card>
  );

  return (
    <View style={styles.container}>
      <FlatList
        data={products}
        renderItem={renderProduct}
        keyExtractor={item => item._id}
        numColumns={2}
      />
    </View>
  );
};

// screens/CheckoutScreen.js
import { CardField, useStripe } from '@stripe/stripe-react-native';

const CheckoutScreen = () => {
  const { initPaymentSheet, presentPaymentSheet } = useStripe();

  const handlePayment = async () => {
    const { error } = await presentPaymentSheet();
    
    if (!error) {
      // Handle successful payment
    }
  };

  return (
    <View style={styles.container}>
      <CardField
        postalCodeEnabled={true}
        style={styles.cardField}
      />
      <Button mode="contained" onPress={handlePayment}>
        Pay Now
      </Button>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: '#f5f5f5'
  },
  card: {
    flex: 1,
    margin: 8
  },
  cardField: {
    height: 50,
    marginVertical: 16
  }
});

export default App;

```

This implementation includes:

1. Backend:
- Express.js server with MongoDB models
- Authentication and order processing
- Stripe payment integration

2. Frontend (Web):
- React app with routing
- Product catalog and cart
- Checkout with Stripe Elements

3. Mobile:
- React Native app with shared business logic
- Native navigation
- Mobile-optimized UI with React Native Paper

To deploy:
1. Set up MongoDB Atlas cluster
2. Deploy backend to AWS EC2/Lambda
3. Deploy web frontend to Vercel/AWS Amplify
4. Release mobile app through App Store/Play Store

Need any specific part explained or modified?
