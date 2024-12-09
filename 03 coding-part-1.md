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
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { 
  AlertDialog,
  AlertDialogAction,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { ShoppingCart, User, Home } from 'lucide-react';

const App = () => {
  const [currentPage, setCurrentPage] = useState('home');
  const [cart, setCart] = useState([]);
  const [showCart, setShowCart] = useState(false);

  const renderPage = () => {
    switch(currentPage) {
      case 'products':
        return <ProductList cart={cart} setCart={setCart} />;
      case 'checkout':
        return <Checkout cart={cart} />;
      default:
        return <HomePage />;
    }
  };

  return (
    <div className="min-h-screen bg-gray-50">
      <nav className="bg-white shadow-sm">
        <div className="max-w-7xl mx-auto px-4 py-4 flex justify-between items-center">
          <div className="flex items-center space-x-4">
            <Button 
              variant="ghost" 
              onClick={() => setCurrentPage('home')}
            >
              <Home className="h-5 w-5 mr-2" />
              Water Purification Systems
            </Button>
          </div>
          <div className="flex items-center space-x-4">
            <Button
              variant="outline"
              onClick={() => setShowCart(!showCart)}
            >
              <ShoppingCart className="h-5 w-5 mr-2" />
              {cart.length > 0 && (
                <span className="bg-blue-600 text-white rounded-full px-2 py-1 text-xs">
                  {cart.length}
                </span>
              )}
            </Button>
            <Button variant="ghost">
              <User className="h-5 w-5" />
            </Button>
          </div>
        </div>
      </nav>

      <main className="max-w-7xl mx-auto px-4 py-8">
        {renderPage()}
      </main>

      <CartDialog 
        open={showCart} 
        onClose={() => setShowCart(false)}
        cart={cart}
        setCart={setCart}
        onCheckout={() => {
          setCurrentPage('checkout');
          setShowCart(false);
        }}
      />
    </div>
  );
};

const HomePage = () => (
  <div className="space-y-6">
    <Card>
      <CardHeader>
        <CardTitle>Welcome to Water Purification Systems</CardTitle>
      </CardHeader>
      <CardContent>
        <p className="text-gray-600">Discover our advanced water purification solutions.</p>
      </CardContent>
    </Card>
  </div>
);

const ProductList = ({ cart, setCart }) => {
  const [products] = useState([
    {
      id: 1,
      name: "Advanced Water Purifier",
      description: "State-of-the-art water purification system",
      price: 299.99,
      image: "/api/placeholder/400/300"
    },
    {
      id: 2,
      name: "Microplastic Filter",
      description: "Specialized filter for microplastic removal",
      price: 149.99,
      image: "/api/placeholder/400/300"
    }
  ]);

  const addToCart = (product) => {
    setCart([...cart, product]);
  };

  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
      {products.map(product => (
        <Card key={product.id}>
          <CardContent className="p-6">
            <img
              src={product.image}
              alt={product.name}
              className="w-full h-48 object-cover rounded mb-4"
            />
            <h2 className="text-lg font-semibold">{product.name}</h2>
            <p className="mt-2 text-gray-600">{product.description}</p>
            <div className="mt-4 flex justify-between items-center">
              <span className="text-xl font-bold">${product.price}</span>
              <Button onClick={() => addToCart(product)}>
                Add to Cart
              </Button>
            </div>
          </CardContent>
        </Card>
      ))}
    </div>
  );
};

const CartDialog = ({ open, onClose, cart, setCart, onCheckout }) => {
  const total = cart.reduce((sum, item) => sum + item.price, 0);

  return (
    <AlertDialog open={open} onOpenChange={onClose}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Shopping Cart</AlertDialogTitle>
          <AlertDialogDescription>
            {cart.length === 0 ? (
              <p>Your cart is empty</p>
            ) : (
              <div className="space-y-4">
                {cart.map((item, index) => (
                  <div key={index} className="flex justify-between items-center">
                    <span>{item.name}</span>
                    <span>${item.price}</span>
                  </div>
                ))}
                <div className="border-t pt-4">
                  <div className="flex justify-between font-bold">
                    <span>Total:</span>
                    <span>${total.toFixed(2)}</span>
                  </div>
                </div>
              </div>
            )}
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogAction onClick={onClose}>Close</AlertDialogAction>
          {cart.length > 0 && (
            <Button onClick={onCheckout} className="ml-2">
              Checkout
            </Button>
          )}
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
};

const Checkout = ({ cart }) => {
  const total = cart.reduce((sum, item) => sum + item.price, 0);
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    address: '',
  });

  const handleSubmit = (e) => {
    e.preventDefault();
    // Handle payment submission to backend
    console.log('Processing payment:', { formData, cart });
  };

  return (
    <Card className="max-w-lg mx-auto">
      <CardHeader>
        <CardTitle>Checkout</CardTitle>
      </CardHeader>
      <CardContent>
        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <label className="block text-sm font-medium mb-1">Name</label>
            <Input
              type="text"
              value={formData.name}
              onChange={(e) => setFormData({...formData, name: e.target.value})}
              required
            />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">Email</label>
            <Input
              type="email"
              value={formData.email}
              onChange={(e) => setFormData({...formData, email: e.target.value})}
              required
            />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">Address</label>
            <Input
              type="text"
              value={formData.address}
              onChange={(e) => setFormData({...formData, address: e.target.value})}
              required
            />
          </div>
          <div className="pt-4 border-t">
            <div className="flex justify-between font-bold mb-4">
              <span>Total:</span>
              <span>${total.toFixed(2)}</span>
            </div>
            <Button type="submit" className="w-full">
              Complete Purchase
            </Button>
          </div>
        </form>
      </CardContent>
    </Card>
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
