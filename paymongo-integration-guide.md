# PayMongo Integration Guide for Shop & Drop

## Overview
This guide provides comprehensive instructions for integrating PayMongo payment processing into your Shop & Drop mobile application. PayMongo is a leading payment gateway in the Philippines that supports various payment methods including credit cards, debit cards, GCash, PayMaya, and bank transfers.

## Table of Contents
1. [Getting Started](#getting-started)
2. [Account Setup](#account-setup)
3. [API Integration](#api-integration)
4. [Payment Methods](#payment-methods)
5. [Security Implementation](#security-implementation)
6. [Testing](#testing)
7. [Production Deployment](#production-deployment)
8. [Troubleshooting](#troubleshooting)

## Getting Started

### Prerequisites
- Valid business registration in the Philippines
- Bank account for settlement
- Valid mobile application (Android/iOS)
- SSL certificate for secure communication

### PayMongo Account Registration
1. Visit [PayMongo Dashboard](https://dashboard.paymongo.com/)
2. Click "Sign Up" and create your merchant account
3. Complete the business verification process
4. Submit required documents:
   - Business registration certificate
   - Bank account details
   - Valid ID of business owner
   - Proof of address

## Account Setup

### 1. Dashboard Configuration
```bash
# Access your PayMongo dashboard
https://dashboard.paymongo.com/login

# Navigate to Settings > API Keys
# Copy your Public Key and Secret Key
```

### 2. Environment Configuration
```javascript
// config/paymongo.js
const config = {
  development: {
    publicKey: 'pk_test_your_public_key_here',
    secretKey: 'sk_test_your_secret_key_here',
    webhookSecret: 'whsec_your_webhook_secret_here',
    baseUrl: 'https://api.paymongo.com/v1'
  },
  production: {
    publicKey: 'pk_live_your_public_key_here',
    secretKey: 'sk_live_your_secret_key_here',
    webhookSecret: 'whsec_your_webhook_secret_here',
    baseUrl: 'https://api.paymongo.com/v1'
  }
};

module.exports = config;
```

## API Integration

### 1. Payment Intent Creation
```javascript
// services/paymentService.js
const axios = require('axios');
const config = require('../config/paymongo');

class PaymentService {
  constructor() {
    this.baseUrl = config.baseUrl;
    this.secretKey = config.secretKey;
  }

  async createPaymentIntent(amount, currency = 'PHP', metadata = {}) {
    try {
      const response = await axios.post(
        `${this.baseUrl}/payment_intents`,
        {
          data: {
            attributes: {
              amount: amount * 100, // Convert to centavos
              currency: currency,
              metadata: metadata
            }
          }
        },
        {
          headers: {
            'Authorization': `Basic ${Buffer.from(this.secretKey + ':').toString('base64')}`,
            'Content-Type': 'application/json'
          }
        }
      );
      
      return response.data.data;
    } catch (error) {
      throw new Error(`Payment intent creation failed: ${error.response?.data?.errors?.[0]?.detail || error.message}`);
    }
  }
}

module.exports = PaymentService;
```

### 2. Payment Method Attachment
```javascript
// services/paymentService.js (continued)
async attachPaymentMethod(paymentIntentId, paymentMethodId) {
  try {
    const response = await axios.post(
      `${this.baseUrl}/payment_intents/${paymentIntentId}/attach`,
      {
        data: {
          attributes: {
            payment_method: paymentMethodId
          }
        }
      },
      {
        headers: {
          'Authorization': `Basic ${Buffer.from(this.secretKey + ':').toString('base64')}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    return response.data.data;
  } catch (error) {
    throw new Error(`Payment method attachment failed: ${error.response?.data?.errors?.[0]?.detail || error.message}`);
  }
}
```

### 3. Payment Confirmation
```javascript
// services/paymentService.js (continued)
async confirmPayment(paymentIntentId) {
  try {
    const response = await axios.post(
      `${this.baseUrl}/payment_intents/${paymentIntentId}/confirm`,
      {},
      {
        headers: {
          'Authorization': `Basic ${Buffer.from(this.secretKey + ':').toString('base64')}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    return response.data.data;
  } catch (error) {
    throw new Error(`Payment confirmation failed: ${error.response?.data?.errors?.[0]?.detail || error.message}`);
  }
}
```

## Payment Methods

### 1. Credit/Debit Cards
```javascript
// components/PaymentForm.jsx
import React, { useState } from 'react';
import { CardElement, useStripe, useElements } from '@stripe/react-stripe-js';

const PaymentForm = ({ amount, onSuccess, onError }) => {
  const stripe = useStripe();
  const elements = useElements();
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (event) => {
    event.preventDefault();
    
    if (!stripe || !elements) return;

    setLoading(true);

    try {
      // Create payment method
      const { error, paymentMethod } = await stripe.createPaymentMethod({
        type: 'card',
        card: elements.getElement(CardElement),
      });

      if (error) {
        onError(error.message);
        return;
      }

      // Create payment intent
      const response = await fetch('/api/create-payment-intent', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          amount: amount,
          payment_method_id: paymentMethod.id
        })
      });

      const { client_secret } = await response.json();

      // Confirm payment
      const { error: confirmError } = await stripe.confirmCardPayment(client_secret);
      
      if (confirmError) {
        onError(confirmError.message);
      } else {
        onSuccess();
      }
    } catch (error) {
      onError('Payment failed. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <CardElement
        options={% raw %}{{
          style: {
            base: {
              fontSize: '16px',
              color: '#424770',
              '::placeholder': {
                color: '#aab7c4',
              },
            },
          },
        }}{% endraw %}
      />
      <button disabled={!stripe || loading}>
        {loading ? 'Processing...' : `Pay ₱${amount}`}
      </button>
    </form>
  );
};
```

### 2. GCash Integration
```javascript
// services/gcashService.js
class GCashService {
  async createGCashPayment(amount, returnUrl, cancelUrl) {
    try {
      const response = await axios.post(
        `${this.baseUrl}/sources`,
        {
          data: {
            attributes: {
              type: 'gcash',
              amount: amount * 100,
              currency: 'PHP',
              redirect: {
                success: returnUrl,
                failed: cancelUrl
              }
            }
          }
        },
        {
          headers: {
            'Authorization': `Basic ${Buffer.from(this.secretKey + ':').toString('base64')}`,
            'Content-Type': 'application/json'
          }
        }
      );
      
      return response.data.data;
    } catch (error) {
      throw new Error(`GCash payment creation failed: ${error.response?.data?.errors?.[0]?.detail || error.message}`);
    }
  }
}
```

### 3. PayMaya Integration
```javascript
// services/paymayaService.js
class PayMayaService {
  async createPayMayaPayment(amount, returnUrl, cancelUrl) {
    try {
      const response = await axios.post(
        `${this.baseUrl}/sources`,
        {
          data: {
            attributes: {
              type: 'paymaya',
              amount: amount * 100,
              currency: 'PHP',
              redirect: {
                success: returnUrl,
                failed: cancelUrl
              }
            }
          }
        },
        {
          headers: {
            'Authorization': `Basic ${Buffer.from(this.secretKey + ':').toString('base64')}`,
            'Content-Type': 'application/json'
          }
        }
      );
      
      return response.data.data;
    } catch (error) {
      throw new Error(`PayMaya payment creation failed: ${error.response?.data?.errors?.[0]?.detail || error.message}`);
    }
  }
}
```

## Security Implementation

### 1. Webhook Verification
```javascript
// middleware/webhookVerification.js
const crypto = require('crypto');

const verifyWebhook = (req, res, next) => {
  const signature = req.headers['paymongo-signature'];
  const webhookSecret = process.env.PAYMONGO_WEBHOOK_SECRET;
  
  if (!signature || !webhookSecret) {
    return res.status(400).json({ error: 'Missing signature or webhook secret' });
  }

  const expectedSignature = crypto
    .createHmac('sha256', webhookSecret)
    .update(JSON.stringify(req.body))
    .digest('hex');

  if (signature !== expectedSignature) {
    return res.status(400).json({ error: 'Invalid signature' });
  }

  next();
};

module.exports = verifyWebhook;
```

### 2. Webhook Handler
```javascript
// routes/webhooks.js
const express = require('express');
const router = express.Router();
const verifyWebhook = require('../middleware/webhookVerification');

router.post('/paymongo', verifyWebhook, async (req, res) => {
  try {
    const { data } = req.body;
    const eventType = data.attributes.type;

    switch (eventType) {
      case 'payment.paid':
        await handlePaymentSuccess(data);
        break;
      case 'payment.failed':
        await handlePaymentFailure(data);
        break;
      case 'source.chargeable':
        await handleSourceChargeable(data);
        break;
      default:
        console.log(`Unhandled event type: ${eventType}`);
    }

    res.status(200).json({ received: true });
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).json({ error: 'Webhook processing failed' });
  }
});

async function handlePaymentSuccess(data) {
  // Update order status to paid
  // Send confirmation email
  // Trigger delivery process
  console.log('Payment successful:', data);
}

async function handlePaymentFailure(data) {
  // Update order status to failed
  // Send failure notification
  // Release inventory
  console.log('Payment failed:', data);
}

module.exports = router;
```

## Testing

### 1. Test Cards
```javascript
// test/testCards.js
const testCards = {
  visa: {
    number: '4242424242424242',
    expiry: '12/25',
    cvc: '123'
  },
  visaDebit: {
    number: '4000056655665556',
    expiry: '12/25',
    cvc: '123'
  },
  mastercard: {
    number: '5555555555554444',
    expiry: '12/25',
    cvc: '123'
  },
  declined: {
    number: '4000000000000002',
    expiry: '12/25',
    cvc: '123'
  },
  insufficientFunds: {
    number: '4000000000009995',
    expiry: '12/25',
    cvc: '123'
  }
};

module.exports = testCards;
```

### 2. Test Environment Setup
```javascript
// config/test.js
const testConfig = {
  publicKey: 'pk_test_your_test_public_key',
  secretKey: 'sk_test_your_test_secret_key',
  webhookSecret: 'whsec_your_test_webhook_secret',
  baseUrl: 'https://api.paymongo.com/v1'
};

module.exports = testConfig;
```

## Production Deployment

### 1. Environment Variables
```bash
# .env.production
PAYMONGO_PUBLIC_KEY=pk_live_your_live_public_key
PAYMONGO_SECRET_KEY=sk_live_your_live_secret_key
PAYMONGO_WEBHOOK_SECRET=whsec_your_live_webhook_secret
PAYMONGO_BASE_URL=https://api.paymongo.com/v1
```

### 2. SSL Configuration
```nginx
# nginx.conf
server {
    listen 443 ssl;
    server_name your-domain.com;
    
    ssl_certificate /path/to/your/certificate.crt;
    ssl_certificate_key /path/to/your/private.key;
    
    # PayMongo requires TLS 1.2 or higher
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Troubleshooting

### Common Issues

1. **Authentication Errors**
   - Verify API keys are correct
   - Ensure keys match environment (test/live)
   - Check key format and encoding

2. **Payment Failures**
   - Verify amount is in centavos (multiply by 100)
   - Check currency code is 'PHP'
   - Ensure payment method is properly attached

3. **Webhook Issues**
   - Verify webhook URL is accessible
   - Check signature verification
   - Ensure webhook secret is correct

4. **SSL/TLS Issues**
   - Verify SSL certificate is valid
   - Ensure TLS 1.2+ is supported
   - Check certificate chain is complete

### Error Codes
```javascript
// common error codes
const errorCodes = {
  'authentication_required': 'Invalid or missing API key',
  'invalid_request_error': 'Request parameters are invalid',
  'card_declined': 'Payment card was declined',
  'insufficient_funds': 'Insufficient funds in account',
  'expired_card': 'Payment card has expired',
  'incorrect_cvc': 'CVC code is incorrect',
  'processing_error': 'Payment processing error occurred'
};
```

## Best Practices

1. **Security**
   - Never store card details
   - Use HTTPS for all communications
   - Implement proper error handling
   - Validate all inputs

2. **User Experience**
   - Provide clear error messages
   - Implement loading states
   - Handle network failures gracefully
   - Offer multiple payment methods

3. **Monitoring**
   - Log all payment attempts
   - Monitor webhook deliveries
   - Track payment success rates
   - Set up alerts for failures

## Support and Resources

- **PayMongo Documentation**: https://developers.paymongo.com/
- **API Reference**: https://developers.paymongo.com/reference
- **Support**: support@paymongo.com
- **Status Page**: https://status.paymongo.com/

## Compliance

Ensure your integration complies with:
- PCI DSS requirements
- Data Privacy Act of 2012 (Philippines)
- Bangko Sentral ng Pilipinas regulations
- Local business regulations

---

*This guide is provided as a reference. Always refer to the official PayMongo documentation for the most up-to-date information and best practices.*
