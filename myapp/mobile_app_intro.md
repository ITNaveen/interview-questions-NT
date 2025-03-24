# Even though the user only sees "merkcheckup" in the app UI, the app’s backend code knows the full API URL.
API_BASE_URL = "https://api.merkcheckup.com"


# 1️⃣ User Enters Login Credentials in the Mobile App
Example:
POST https://merkcheckup.com/api/auth/login
Content-Type: application/json
{
  "username": "john.doe",
  "password": "securepassword123"
}
The mobile app needs to find the IP address for merkcheckup.com to send this request.

# 2️⃣ Mobile App DNS Resolution Process
The mobile device first checks its local DNS cache to see if it already knows the IP.
If no cache is found, it queries the Recursive DNS Resolver (provided by ISP, Google 8.8.8.8, or Cloudflare 1.1.1.1).
If the Recursive Resolver doesn’t have it cached, it follows these steps:
Queries the Root DNS Server, which determines the TLD (since .com is used, it directs to the .com TLD server).
The TLD Name Server (for .com) checks where merkcheckup.com is registered and finds Route 53 as the authoritative DNS.
The Recursive Resolver asks Route 53 for the IP of merkcheckup.com.
Route 53 returns the IP address of AWS API Gateway, which handles authentication.

# 3. based on frontend code app has already configured api gatway.
- ip and creds go to api gateway.
- API Gateway is configured in AWS to receive traffic on https://api.merkcheckup.com.
- It acts as the entry point and forwards the request to the correct backend microservice.

# 4. creds along with IP gets validated - 
1️⃣ API Gateway forwards the login request (email/password) to AWS Cognito (instead of a custom Auth Service).
2️⃣ Cognito checks credentials against its User Pool (AWS-managed user directory).
3️⃣ If credentials are valid, Cognito generates a JWT token (ID Token, Access Token, Refresh Token).
4️⃣ Cognito returns the JWT tokens to API Gateway.
5️⃣ API Gateway forwards the JWT back to the user’s mobile app.

# 5. user gets jwt token and then raise a request.
1. user send request that with header of that request containing token.
2. api gateway receives this and validate and then forward this to ALB and rest is same as web app.