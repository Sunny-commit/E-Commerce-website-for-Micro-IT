# 🌐 E-Commerce Website - Full-Stack Web Development

A **complete e-commerce platform** with product catalog, shopping cart, payment integration, and admin dashboard.

## 🎯 Overview

This project provides:
- ✅ Product management
- ✅ Shopping cart system
- ✅ Payment processing
- ✅ User authentication
- ✅ Order management
- ✅ Admin dashboard
- ✅ Real-time inventory

## 🛍️ Product Management

```python
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_cors import CORS
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///ecommerce.db'
db = SQLAlchemy(app)
CORS(app)

class Product(db.Model):
    """Product model"""
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text)
    price = db.Column(db.Float, nullable=False)
    stock = db.Column(db.Integer, default=0)
    category = db.Column(db.String(50))
    image_url = db.Column(db.String(255))
    rating = db.Column(db.Float, default=0)
    reviews_count = db.Column(db.Integer, default=0)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    def to_dict(self):
        return {
            'id': self.id,
            'name': self.name,
            'description': self.description,
            'price': self.price,
            'stock': self.stock,
            'category': self.category,
            'image_url': self.image_url,
            'rating': self.rating,
            'reviews_count': self.reviews_count
        }

@app.route('/api/products', methods=['GET'])
def get_products():
    """List products with filters"""
    category = request.args.get('category')
    sort_by = request.args.get('sort', 'created_at')
    page = request.args.get('page', 1, type=int)
    
    query = Product.query
    
    if category:
        query = query.filter_by(category=category)
    
    products = query.order_by(sort_by).paginate(page=page, per_page=20)
    
    return jsonify({
        'products': [p.to_dict() for p in products.items],
        'total': products.total,
        'pages': products.pages,
        'page': page
    })

@app.route('/api/products/<int:product_id>', methods=['GET'])
def get_product(product_id):
    """Get product details"""
    product = Product.query.get_or_404(product_id)
    return jsonify(product.to_dict())

@app.route('/api/products', methods=['POST'])
def create_product():
    """Admin: Create product"""
    data = request.get_json()
    
    product = Product(
        name=data['name'],
        description=data.get('description'),
        price=data['price'],
        stock=data.get('stock', 0),
        category=data.get('category'),
        image_url=data.get('image_url')
    )
    
    db.session.add(product)
    db.session.commit()
    
    return jsonify(product.to_dict()), 201

@app.route('/api/products/<int:product_id>', methods=['PUT'])
def update_product(product_id):
    """Admin: Update product"""
    product = Product.query.get_or_404(product_id)
    data = request.get_json()
    
    product.name = data.get('name', product.name)
    product.price = data.get('price', product.price)
    product.stock = data.get('stock', product.stock)
    product.description = data.get('description', product.description)
    
    db.session.commit()
    
    return jsonify(product.to_dict())
```

## 🛒 Shopping Cart

```python
class Cart(db.Model):
    """Shopping cart"""
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    items = db.relationship('CartItem', backref='cart', cascade='all, delete-orphan')

class CartItem(db.Model):
    """Individual cart item"""
    id = db.Column(db.Integer, primary_key=True)
    cart_id = db.Column(db.Integer, db.ForeignKey('cart.id'))
    product_id = db.Column(db.Integer, db.ForeignKey('product.id'))
    quantity = db.Column(db.Integer, default=1)
    product = db.relationship('Product')

@app.route('/api/cart', methods=['GET'])
def get_cart():
    """Get current user cart"""
    user_id = get_current_user_id()
    cart = Cart.query.filter_by(user_id=user_id).first_or_404()
    
    total = sum(item.product.price * item.quantity for item in cart.items)
    
    return jsonify({
        'id': cart.id,
        'items': [{
            'product_id': item.product_id,
            'name': item.product.name,
            'price': item.product.price,
            'quantity': item.quantity,
            'total': item.product.price * item.quantity
        } for item in cart.items],
        'total': total,
        'item_count': len(cart.items)
    })

@app.route('/api/cart/add', methods=['POST'])
def add_to_cart():
    """Add item to cart"""
    user_id = get_current_user_id()
    data = request.get_json()
    
    cart = Cart.query.filter_by(user_id=user_id).first()
    if not cart:
        cart = Cart(user_id=user_id)
        db.session.add(cart)
        db.session.flush()
    
    product = Product.query.get_or_404(data['product_id'])
    
    # Check stock
    if product.stock < data.get('quantity', 1):
        return jsonify({'error': 'Insufficient stock'}), 400
    
    # Add to cart
    cart_item = CartItem.query.filter_by(
        cart_id=cart.id,
        product_id=data['product_id']
    ).first()
    
    if cart_item:
        cart_item.quantity += data.get('quantity', 1)
    else:
        cart_item = CartItem(
            cart_id=cart.id,
            product_id=data['product_id'],
            quantity=data.get('quantity', 1)
        )
        db.session.add(cart_item)
    
    db.session.commit()
    
    return jsonify({'message': 'Added to cart'}), 200

@app.route('/api/cart/item/<int:item_id>', methods=['DELETE'])
def remove_from_cart(item_id):
    """Remove from cart"""
    item = CartItem.query.get_or_404(item_id)
    db.session.delete(item)
    db.session.commit()
    
    return jsonify({'message': 'Removed from cart'})
```

## 💳 Payment Integration

```python
import stripe

stripe.api_key = 'your-stripe-key'

class Order(db.Model):
    """Order model"""
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer)
    total_amount = db.Column(db.Float)
    status = db.Column(db.String(20), default='pending')
    payment_id = db.Column(db.String(100))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    items = db.relationship('OrderItem', backref='order', cascade='all, delete-orphan')

class OrderItem(db.Model):
    """Items in order"""
    id = db.Column(db.Integer, primary_key=True)
    order_id = db.Column(db.Integer, db.ForeignKey('order.id'))
    product_id = db.Column(db.Integer)
    quantity = db.Column(db.Integer)
    price = db.Column(db.Float)

@app.route('/api/checkout', methods=['POST'])
def checkout():
    """Create order and payment"""
    data = request.get_json()
    user_id = get_current_user_id()
    cart = Cart.query.filter_by(user_id=user_id).first_or_404()
    
    # Calculate total
    total = sum(item.product.price * item.quantity for item in cart.items)
    
    # Create payment intent
    intent = stripe.PaymentIntent.create(
        amount=int(total * 100),
        currency='usd',
        payment_method_types=['card']
    )
    
    # Create order
    order = Order(
        user_id=user_id,
        total_amount=total,
        status='processing'
    )
    
    for item in cart.items:
        order_item = OrderItem(
            product_id=item.product_id,
            quantity=item.quantity,
            price=item.product.price
        )
        order.items.append(order_item)
        
        # Update stock
        product = Product.query.get(item.product_id)
        product.stock -= item.quantity
    
    db.session.add(order)
    db.session.commit()
    
    return jsonify({
        'order_id': order.id,
        'client_secret': intent.client_secret,
        'amount': total
    })

@app.route('/api/payment/confirm', methods=['POST'])
def confirm_payment():
    """Confirm payment completion"""
    data = request.get_json()
    
    order = Order.query.get_or_404(data['order_id'])
    order.status = 'confirmed'
    order.payment_id = data['payment_id']
    
    db.session.commit()
    
    return jsonify({'message': 'Payment confirmed'})
```

## 📊 Order Management

```python
@app.route('/api/orders', methods=['GET'])
def get_user_orders():
    """User orders"""
    user_id = get_current_user_id()
    orders = Order.query.filter_by(user_id=user_id).all()
    
    return jsonify([{
        'id': o.id,
        'total': o.total_amount,
        'status': o.status,
        'created_at': o.created_at.isoformat(),
        'items_count': len(o.items)
    } for o in orders])

@app.route('/api/orders/<int:order_id>', methods=['GET'])
def get_order_details(order_id):
    """Order details"""
    order = Order.query.get_or_404(order_id)
    
    return jsonify({
        'id': order.id,
        'total': order.total_amount,
        'status': order.status,
        'items': [{
            'product_id': item.product_id,
            'quantity': item.quantity,
            'price': item.price,
            'total': item.quantity * item.price
        } for item in order.items]
    })
```

## 💡 Interview Talking Points

**Q: Shopping cart design?**
```
Answer:
- Session-based (simple, stateless)
- Database-based (persistent)
- Redis cache (fast, scale)
- Wish list similar structure
```

**Q: Payment security?**
```
Answer:
- Never store card numbers
- Use payment processor (Stripe, PayPal)
- PCI compliance required
- Encrypted communication (HTTPS)
- Tokenization for recurring
```

## 🌟 Portfolio Value

✅ Full-stack development
✅ Database design
✅ Payment integration
✅ Inventory management
✅ REST API design
✅ E-commerce domain
✅ Scalable architecture

---

**Technologies**: Flask, SQLAlchemy, Stripe, PostgreSQL

