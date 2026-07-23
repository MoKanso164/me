// /api/send-email.js
// Vercel Serverless Function (Node.js runtime).
// Receives the contact form payload, validates it, and sends an email
// via the Resend API. The Resend API key never touches the browser —
// it is read from a server-side environment variable.

const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

// Very small in-memory rate limiter. Serverless instances are ephemeral
// and can be scaled to zero, so this is a best-effort spam brake, not a
// guarantee. It stops the same instance from being hammered in a burst.
const recentSubmissions = new Map();
const RATE_LIMIT_WINDOW_MS = 60 * 1000; // 1 minute
const RATE_LIMIT_MAX = 3; // max submissions per IP per window

function isRateLimited(ip) {
  const now = Date.now();
  const entry = recentSubmissions.get(ip);
  if (!entry || now - entry.windowStart > RATE_LIMIT_WINDOW_MS) {
    recentSubmissions.set(ip, { windowStart: now, count: 1 });
    return false;
  }
  entry.count += 1;
  return entry.count > RATE_LIMIT_MAX;
}

function escapeHtml(str) {
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}

module.exports = async function handler(req, res) {
  if (req.method !== 'POST') {
    res.setHeader('Allow', 'POST');
    return res.status(405).json({ success: false, error: 'Method not allowed.' });
  }

  // ---- Read server-side config ----
  const RESEND_API_KEY = process.env.RESEND_API_KEY;
  const TO_EMAIL = process.env.CONTACT_TO_EMAIL;
  const FROM_EMAIL = process.env.CONTACT_FROM_EMAIL; // must be a Resend-verified sender/domain

  if (!RESEND_API_KEY || !TO_EMAIL || !FROM_EMAIL) {
    console.error('Missing required environment variables for email sending.');
    return res.status(500).json({
      success: false,
      error: 'Email service is not configured. Please contact the site owner directly.'
    });
  }

  // ---- Identify the visitor (server-side, cannot be spoofed by the page) ----
  const forwardedFor = req.headers['x-forwarded-for'];
  const ip =
    (typeof forwardedFor === 'string' ? forwardedFor.split(',')[0].trim() : null) ||
    req.socket?.remoteAddress ||
    'Unknown';
  const serverUserAgent = req.headers['user-agent'] || 'Unknown';

  // ---- Basic spam brake ----
  if (isRateLimited(ip)) {
    return res.status(429).json({ success: false, error: 'Too many requests. Please try again later.' });
  }

  const body = req.body || {};
  const name = typeof body.name === 'string' ? body.name.trim() : '';
  const email = typeof body.email === 'string' ? body.email.trim() : '';
  const subject = typeof body.subject === 'string' ? body.subject.trim() : '';
  const message = typeof body.message === 'string' ? body.message.trim() : '';
  const phone = typeof body.phone === 'string' ? body.phone.trim() : '';
  const website = typeof body.website === 'string' ? body.website.trim() : ''; // honeypot
  const clientTime = typeof body.clientTime === 'string' ? body.clientTime : '';
  const clientUserAgent = typeof body.userAgent === 'string' ? body.userAgent : serverUserAgent;

  // ---- Honeypot check: real visitors never fill this hidden field ----
  if (website) {
    // Pretend success so bots don't learn to avoid the trap; don't send an email.
    return res.status(200).json({ success: true });
  }

  // ---- Validation ----
  if (!name || !email || !message) {
    return res.status(400).json({ success: false, error: 'Name, email, and message are required.' });
  }
  if (!EMAIL_REGEX.test(email)) {
    return res.status(400).json({ success: false, error: 'Please provide a valid email address.' });
  }
  if (message.length < 2 || message.length > 5000) {
    return res.status(400).json({ success: false, error: 'Message must be between 2 and 5000 characters.' });
  }
  if (name.length > 200 || subject.length > 300) {
    return res.status(400).json({ success: false, error: 'One of the fields is too long.' });
  }

  const submittedAt = new Date().toISOString();

  const htmlBody = `
    <div style="font-family: Arial, sans-serif; font-size: 14px; color: #111;">
      <h2 style="margin-bottom: 12px;">New contact form submission</h2>
      <table cellpadding="6" cellspacing="0" style="border-collapse: collapse;">
        <tr><td><strong>Name</strong></td><td>${escapeHtml(name)}</td></tr>
        <tr><td><strong>Email</strong></td><td>${escapeHtml(email)}</td></tr>
        <tr><td><strong>Subject</strong></td><td>${escapeHtml(subject || '(none provided)')}</td></tr>
        <tr><td><strong>Phone</strong></td><td>${escapeHtml(phone || '(none provided)')}</td></tr>
        <tr><td><strong>Submitted (server time, UTC)</strong></td><td>${escapeHtml(submittedAt)}</td></tr>
        <tr><td><strong>Submitted (visitor's local time)</strong></td><td>${escapeHtml(clientTime || 'Unknown')}</td></tr>
        <tr><td><strong>IP address</strong></td><td>${escapeHtml(ip)}</td></tr>
        <tr><td><strong>User agent</strong></td><td>${escapeHtml(clientUserAgent)}</td></tr>
      </table>
      <p style="margin-top: 16px;"><strong>Message:</strong></p>
      <p style="white-space: pre-wrap; border-left: 3px solid #ccc; padding-left: 12px;">${escapeHtml(message)}</p>
    </div>
  `;

  const textBody =
    `New contact form submission\n\n` +
    `Name: ${name}\n` +
    `Email: ${email}\n` +
    `Subject: ${subject || '(none provided)'}\n` +
    `Phone: ${phone || '(none provided)'}\n` +
    `Submitted (server time, UTC): ${submittedAt}\n` +
    `Submitted (visitor's local time): ${clientTime || 'Unknown'}\n` +
    `IP address: ${ip}\n` +
    `User agent: ${clientUserAgent}\n\n` +
    `Message:\n${message}\n`;

  try {
    const resendRes = await fetch('https://api.resend.com/emails', {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${RESEND_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        from: FROM_EMAIL, // e.g. "Portfolio Contact Form <contact@yourdomain.com>"
        to: [TO_EMAIL],
        reply_to: email, // lets you hit "Reply" and answer the visitor directly
        subject: subject ? `[Portfolio] ${subject}` : `[Portfolio] New message from ${name}`,
        html: htmlBody,
        text: textBody
      })
    });

    if (!resendRes.ok) {
      const errText = await resendRes.text();
      console.error('Resend API error:', resendRes.status, errText);
      return res.status(502).json({ success: false, error: 'Email service rejected the message. Please try again later.' });
    }

    return res.status(200).json({ success: true });
  } catch (err) {
    console.error('Failed to send email:', err);
    return res.status(500).json({ success: false, error: 'Unexpected server error while sending your message.' });
  }
};
