# Notes

## AWS

1. Create an AWS account.
2. Inside the AWS console:
   1. Navigate to 'IAM'.
   2. Create a user with appropriate permissions.
   3. In 'Security credentials', create an 'Access Key'.
3. Run `aws configure` in your terminal and complete the steps.
4. Check the `.aws` folder in the root of your PC.
5. Create an EC2 instance:
   1. Add the API's listening port to the 'Security Group' of your EC2 instance.
6. Connect to the EC2 instance using `ssh -i /path/to/your/private/key.pem <user>@<your-instance-ip>`.
   1. Remember to update the OS (`sudo apt update`).
   2. Install all the necessary service programs.
7. Start Docker:
   - `sudo service docker start`.
   - `sudo docker network create {network}`.
   - `sudo docker run --network {network} -d {docker}`.
   - `sudo docker-compose up -d`
8. Install Nginx.
9. Set proper permissions for the Nginx folder:
   - `sudo chown -R $USER:$USER /var/www/html/`.
   - `sudo chmod -R 755 /var/www/html/`.
10. Install snapd: `sudo apt install snapd`.
11. Check snapd version: `sudo snap install core; sudo snap refresh core`.
12. Remove Certbot: `sudo apt-get remove certbot`.
13. Install Certbot with snap: `sudo snap install --classic certbot`.
14. (Optional) Create 'Certbot' shortcut: `sudo ln -s /snap/bin/certbot /usr/bin/certbot`.
15. Check Certbot version: `sudo certbot --version` or `sudo /snap/bin/certbot --version`.
16. Run Certbot with Nginx: `sudo certbot --nginx` or `sudo /snap/bin/certbot --nginx`.

## Certbot Renew

To renew your Let's Encrypt certificate for the domain "benjaminpla.com" and its subdomain "www.benjaminpla.com," you can follow these general steps:

1. **Check Your Current Setup**: Make sure you have a web server (such as Apache, Nginx, etc.) running and hosting your website on the mentioned domain names.

2. **Update Certbot (if used)**: If you've been using Certbot to manage your certificates, ensure that it's up to date. You can do this with the following commands:

   ```
   sudo apt update
   sudo apt install --only-upgrade certbot
   ```

3. **Renew the Certificate**: To renew the certificate, you can run the Certbot renewal command. It's recommended to include the `--dry-run` option to test the renewal process without actually renewing the certificate. This can help you identify any potential issues beforehand.

   ```
   sudo certbot renew --dry-run
   ```

   If the dry run is successful and doesn't show any errors, you can then run the command without `--dry-run` to actually renew the certificate:

   ```
   sudo certbot renew
   ```

   Certbot will automatically renew certificates that are close to expiration.

4. **Automatic Renewal**: To avoid manual renewal in the future, set up a cron job to run the `certbot renew` command periodically. This ensures that your certificates are always up to date. Certbot often adds a cron job for you during installation, but you should check to make sure it's in place:

   ```
   sudo crontab -l
   ```

   If the cron job isn't present, you can add it manually. Open the cron table for editing:

   ```
   sudo crontab -e
   ```

   Then add the following line to run the renewal check every day:

   ```
   0 0 * * * /usr/bin/certbot renew
   ```

   Save and exit the editor.

5. **Check Renewal Status**: After running the renewal command, check if your certificates were successfully renewed:

   ```
   sudo certbot certificates
   ```

   You should see an output that confirms the new expiration date for your certificates.

6. **Restart Web Server**: Sometimes, after certificate renewal, you might need to restart your web server to load the new certificates:

   ```
   sudo systemctl restart apache2   # For Apache
   sudo systemctl restart nginx     # For Nginx
   ```

Remember that specific steps can vary based on your server setup and configuration. If you encounter any issues or have a different setup, you might need to adjust the steps accordingly. Always refer to the documentation of your web server software and Certbot for accurate and up-to-date instructions.

## Usefull Things

- Run Docker without 'sudo': `sudo usermod -aG docker $USER`.
- Start Docker on system boot: `sudo chkconfig docker on`.
- Docker volumes `-v {hostPath}:{dockerPath}:ro`.
- Sync S3 bucket `aws s3 sync s3://<s3_bucket> <local_folder>`

# OpenID Connect (OIDC)

> Focuses on authentication

## Terminology

Relying Party (RP) => abc.com

Identity Provider (IdP) => google.com

## Flow

- user want to log in abc.com
- abc.com redirects user to google.com's OIDC endpoint with parameters
  - response_type: ['code', 'token', 'id_token']
  - client_id
  - redirect_uri
  - scope
  - state: random value to prevent CSRF
  - nonce: random to prevent OIDC attacks
  - prompt: ['none', 'login', 'consent']
  - max_age
  - acr_values: MFA
  - custom_parameters
- user logs in google.com
- google.com consent page
- google.com generate a token
- google.com redirects user to abc.com with the token
- abc.com validates token and extract info from it
- user is logged in abc.com using google.com

## Notes

- abc.com can rely only on it's IdPs, without its own user database _(not recommended)_

- abc.com can log in users with google.com and add them in its own user database.Then give them the possibility to change their loca password or unlik google.com. _(hybrid - recommended)_

### Real Scenario

- I created an AAD
- it gives you POST /authorization, POST /token and more endpoints, client_id and tenant_id
- in Postman I created the full endpoint with the required query params _(there are optionals too)_

```
{
    client_id, response_type: 'code', scope: 'openid'
}
```

- using the browser I hit the enpoint provided by Postman _(why I can't do it from Postman or my api?)_
- the browser request calls my redirect*uri api *(POST /callback)\_

```
import express from "express";
import axios from "axios";
import qs from "querystring";

const api = express();
api.use(express.json());

const conf = {
  client_id: "xxx",
  client_secret: "xxx",
  grant_type: "authorization_code",
  redirect_uri: "http://localhost:3000/callback",
  response_type: "code",
  scope: "openid",
  tenant_id: "xxx",
};

api.get("/callback", async (req, res) => {
  const { code } = req?.query;
  const queryParams = qs.stringify(conf);

  const request = await axios.post(
    `https://login.microsoftonline.com/${conf.tenant_id}/oauth2/v2.0/token`,
    qs.stringify({
      grant_type: conf.grant_type,
      client_id: conf.client_id,
      client_secret: conf.client_secret,
      redirect_uri: conf.redirect_uri,
      code,
    }),
    {
      "Content-Type": "application/x-www-form-urlencoded",
    }
  );

  console.log(request.data);
  res.json(request.data);
});

const port = 3000;
api.listen(port, () => console.log(`api on port ${port}`));
```

- I obtained the token
- What happens when I have more than one person, like a group of users in my AAD?

# Open Authorization 2.0 (OAuth 2.0)

> Focuses on authorization

<!-- loading... -->

## Terminology

<!-- loading... -->
Resource Owned (RO) => the user
Client => who wants to data (abc.com)
Authorization Server (AS) => who generate the token (xyz.com || google.com)
Resource Server (RS) => who have the data (xyz.com)

## Flow

- one app (abc.com) wants to obtain data form another app (xyz.com)
<!-- loading... -->
