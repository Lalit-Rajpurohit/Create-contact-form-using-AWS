# AWS Contact Form with Lambda, SES, and Google reCAPTCHA

This project demonstrates how to build a serverless contact form using:

- **Frontend**: HTML + JavaScript
- **Serverless Function**: AWS Lambda (Python)
- **Email Service**: Amazon SES (Simple Email Service)
- **Bot Protection**: Google reCAPTCHA v2
- **Hosting**: AWS Amplify (or S3/CloudFront)

---

## ✨ Features

- Sends contact form data to your email using AWS SES
- Validates the form using Google reCAPTCHA v2
- Prevents spam/bot submissions
- Simple HTML and JavaScript (no frameworks)

---

## 🛠️ Technologies Used

- AWS Lambda (Python)
- AWS API Gateway (REST API)
- AWS SES (Email delivery)
- AWS IAM Role
- Google reCAPTCHA v2
- HTML + JavaScript

---


---

## 🚀 Setup Instructions

### 1. ✅ Create Google reCAPTCHA Keys

1. Go to [Google reCAPTCHA admin](https://www.google.com/recaptcha/admin/create)
2. Choose:
   - **Type**: reCAPTCHA v2 ("I'm not a robot")
   - **Domains**: e.g., `example.com`
3. Save the **Site Key** and **Secret Key**

---

### 2. ✅ Verify Your Domain in Amazon SES

1. Go to the **SES Console** → **Verified Identities**
2. Click **“Create Identity”** → Select **Domain**
3. Enter your domain (e.g., `example.com`)
4. Add the **DNS records (DKIM/CNAME)** in your DNS provider (e.g., Namecheap)
5. Wait for the domain to be **verified**

✅ You can now send email from any address like `noreply@example.com`

---

### 3. 🐍 Create Your Lambda Function (Python)

- Use the `lambda_function.py` file in this repo.
- Add these **environment variables**:

| Key               | Value                  |
|------------------|------------------------|
| `SOURCE_EMAIL`    | `noreply@example.com`  |
| `DEST_EMAIL`      | `your@email.com`       |
| `RECAPTCHA_SECRET`| Your reCAPTCHA secret |

- Set IAM permissions to allow `ses:SendEmail`:

```json
{
  "Effect": "Allow",
  "Action": "ses:SendEmail",
  "Resource": "*"
}
```

Python Lambda Function:
``` Python Lambda Function
import json
import urllib.request
import urllib.parse
import boto3
import os

ses = boto3.client('ses')

def verify_recaptcha(token):
    secret = os.environ['RECAPTCHA_SECRET']
    data = urllib.parse.urlencode({
        'secret': secret,
        'response': token
    }).encode()

    try:
        req = urllib.request.Request("https://www.google.com/recaptcha/api/siteverify", data=data)
        with urllib.request.urlopen(req) as response:
            result = json.loads(response.read().decode())
            return result.get("success", False)
    except Exception as e:
        print("reCAPTCHA verification error:", str(e))
        return False

def lambda_handler(event, context):
    try:
        body = json.loads(event['body'])

        name = body.get('name')
        email = body.get('email')
        message = body.get('message')
        sub = body.get('sub', 'No subject')
        token = body.get('token')

        # Validate reCAPTCHA
        if not token or not verify_recaptcha(token):
            return {
                'statusCode': 400,
                'body': json.dumps({'message': 'reCAPTCHA verification failed'})
            }

        # Validate fields
        if not name or not email or not message:
            return {
                'statusCode': 400,
                'body': json.dumps({'message': 'Missing required fields'})
            }

        # Get source and destination email from environment
        source_email = os.environ['SOURCE_EMAIL']
        dest_email = os.environ['DEST_EMAIL']

        # Send email
        response = ses.send_email(
            Source=source_email,
            Destination={'ToAddresses': [dest_email]},
            Message={
                'Subject': {'Data': f'Contact Form: {sub}'},
                'Body': {
                    'Text': {
                        'Data': f"Name: {name}\nEmail: {email}\nSubject: {sub}\n\nMessage:\n{message}"
                    }
                }
            }
        )

        return {
            'statusCode': 200,
            'body': json.dumps({'message': 'Email sent successfully'})
        }

    except Exception as e:
        print("Error:", str(e))
        return {
            'statusCode': 500,
            'body': json.dumps({'message': 'Server error', 'error': str(e)})
        }
```

4. 🌐 Create API Gateway (REST API)
Create a new API Gateway

Create a POST method /contact

Connect it to your Lambda function

Enable CORS (Allow POST, OPTIONS, headers: Content-Type)

Deploy to a stage (e.g., default)

5. 🧪 Frontend Integration
In index.html:

Add your reCAPTCHA site key

Update the fetch() URL with your API Gateway endpoint

html
```html
<script src="https://www.google.com/recaptcha/api.js" async defer></script>

<form id="contact-form" onsubmit="submitForm(event)">
  <input name="name" required>
  <input name="email" required>
  <textarea name="message" required></textarea>
  <input name="sub" required>
  
  <div class="g-recaptcha" data-sitekey="YOUR_SITE_KEY_HERE"></div>

  <button type="submit">Send</button>
</form>
```

js
```js
<script src="https://www.google.com/recaptcha/api.js" async defer></script>

<form id="contact-form" onsubmit="submitForm(event)">
  <input name="name" required>
  <input name="email" required>
  <textarea name="message" required></textarea>
  <input name="sub" required>
  
  <div class="g-recaptcha" data-sitekey="YOUR_SITE_KEY_HERE"></div>

  <button type="submit">Send</button>
</form>
```

✅ Done!
Your contact form is now:

Serverless

Spam-resistant

Integrated with SES + reCAPTCHA

Ready to use on your own domain

🛡️ Security Notes
Do not expose your reCAPTCHA secret — keep it in Lambda only

Ensure HTTPS is enabled for API Gateway

Consider logging and rate limiting if going public



