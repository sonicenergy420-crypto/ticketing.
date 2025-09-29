# ticketing.
the project all about selling ticket for events worldwide.

package.json

{
  "name": "ticketing-backend",
  "version": "1.0.0",
  "dependencies": {
    "axios": "^1.6.0",
    "uuid": "^9.0.0",
    "jsonwebtoken": "^9.0.0",
    "qrcode": "^1.5.0",
    "nodemailer": "^6.9.0"
  }
}

vercel.json

{
  "version": 2,
  "builds": [
    { "src": "api/*.js", "use": "@vercel/node" }
  ],
  "routes": [
    { "src": "/api/(.*)", "dest": "/api/$1" }
  ]
}


create-order.js

const { v4: uuidv4 } = require("uuid");

let orders = {}; // temporary in-memory storage

module.exports = async (req, res) => {
  if (req.method !== "POST") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const { eventId, buyerEmail, ticketCount } = req.body;

  if (!eventId || !buyerEmail || !ticketCount) {
    return res.status(400).json({ error: "Missing required fields" });
  }

  const orderId = uuidv4();
  orders[orderId] = {
    orderId,
    eventId,
    buyerEmail,
    ticketCount,
    status: "pending",
  };

  return res.json({ orderId });
};

module.exports.orders = orders;



create-bitpay-invoice.js

const axios = require("axios");

module.exports = async (req, res) => {
  if (req.method !== "POST") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const { orderId, price, currency } = req.body;

  try {
    const response = await axios.post(
      "https://bitpay.com/invoices",
      {
        price,
        currency,
        orderId,
      },
      {
        headers: {
          Authorization: Bearer ${process.env.BITPAY_API_TOKEN},
          "Content-Type": "application/json",
        },
      }
    );

    return res.json(response.data);
  } catch (error) {
    console.error("BitPay error:", error.response?.data || error.message);
    return res.status(500).json({ error: "Failed to create BitPay invoice" });
  }
};



check-order-status.js

const { orders } = require("./create-order");

module.exports = async (req, res) => {
  if (req.method !== "GET") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const { orderId } = req.query;

  if (!orderId || !orders[orderId]) {
    return res.status(404).json({ error: "Order not found" });
  }

  return res.json({ status: orders[orderId].status });
};


webhook-bitpay.js
const { orders } = require("./create-order");
const nodemailer = require("nodemailer");
const QRCode = require("qrcode");

module.exports = async (req, res) => {
  if (req.method !== "POST") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  try {
    const event = req.body;

    if (event.status === "confirmed" && orders[event.orderId]) {
      orders[event.orderId].status = "paid";

      const qrData = JSON.stringify({
        orderId: event.orderId,
        eventId: orders[event.orderId].eventId,
      });
      const qrCode = await QRCode.toDataURL(qrData);

      const transporter = nodemailer.createTransport({
        service: "gmail",
        auth: {
          user: process.env.SMTP_USER || "your.email@gmail.com",
          pass: process.env.SMTP_PASS || "your-app-password",
        },
      });

      await transporter.sendMail({
        from: process.env.SMTP_USER || "your.email@gmail.com",
        to: orders[event.orderId].buyerEmail,
        subject: "Your Event Ticket",
        html: <h1>Ticket Confirmed</h1><p>Scan this QR at the gate.</p><img src="${qrCode}" />,
      });
    }

    res.json({ received: true });
  } catch (error) {
    console.error("Webhook error:", error.message);
    res.status(500).json({ error: "Webhook handling failed" });
  }
};

verify-ticket.js

const jwt = require("jsonwebtoken");

module.exports = async (req, res) => {
  if (req.method !== "POST") {
    return res.status(405).json({ error: "Method not allowed" });
  }

  const { token } = req.body;

  try {
    const decoded = jwt.verify(token, process.env.SERVER_JWT_SECRET);
    return res.json({ valid: true, data: decoded });
  } catch (error) {
    return res.status(400).json({ valid: false, error: "Invalid ticket" });
  }
};



health.js

module.exports = async (req, res) => {
  return res.json({ status: "ok", timestamp: Date.now() });
};





