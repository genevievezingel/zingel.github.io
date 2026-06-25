---
layout: post
title: "Stop hackers stealing Credit Card numbers from your Website"
description: "Having users manage their own payment details and having automated the billing is a huge time saver and it’s expected of websites. Though with anything to do with money, this needs to be done securely otherwise it will end in tears. Consider using the solutions adopted by the payment gateways Stripe and Braintree to secure your customer credit cards. This blog looks into how this works and what are its limitations"
date: 2017-10-26
read: "8 min read"
category: software
permalink: /blog/pci_stripe/
---

Having users manage their own payment details and having automated the billing is a huge time saver and it’s expected of websites. Though with anything to do with money, this needs to be done securely otherwise it will end in tears. Consider using the solutions adopted by the payment gateways Stripe and Braintree to secure your customer credit cards. This blog looks into how this works and what are its limitations.
<img src="{{ '/assets/images/blog/PCI_stripe/figure1.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">


The solution is to make sure your website never records your customer’s credit card details. If the website never has the card information, the customer card information cannot be stolen. 

If you are wondering how these gateways prevent your website from seeing or storing the credit cards, since it looks like the credit card number is entered into the site, I am glad you asked. 
 
I will use Stripe’s checkout form in my example (stripe.js).
### Customer Steps:

1. Customer wants to buy the service
2. They go to the payment page and click the payment button. This displays a modal checkout form
3. Customer enters their credit card details
4. Stripe checks their credit card and either: a) display  errors or b) returns a token that is stored in a hidden form on the payment page before being sent to the backend server. 
5. This token and Stripe’s secret key allows the backend to charge the client

Let’s go through the steps While step (1) is awesome - making those sales - the story begins at step (2) “The checkout form”.

The checkout form can be added to the payment page using code supplied by Stripe. This code has several configurable options (see appendix) and must include the public/publishable key that identifies the account on Stripe.

When the Javascript completes on the payment page, the resulting page includes a hidden form (in red) and an iFrame that contains the credit card form. 
<img src="{{ '/assets/images/blog/PCI_stripe/iframe_small.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">

The iFrame (in blue) that contains the credit card form is sandboxed from the rest of the website since it has a different source to the website domain (Origin Policy). This prevents any malicious scripts on the website from stealing credit card info from the iFrame.

This will make more sense knowing how iFrame works. iFrame inserts another HTML page into the current page. They act like windows allowing the webpage to view another url. In the payment form, the iFrame displays an url on Stripe that hosts the credit card form. It may look like the form is part of the payment page but it’s hosted by Stripe. 

The customer(3)  enters their credit card details into Stripe’s embedded form. The credit card’s checks are handled on Stripe (4). Any validation issues are displayed on the credit form (4a). 

If the card check passes, a token passes from the iFrame to the payment page using the protocol (postMessage). This protocol allows messages to be passed between iFrames/Windows; a token is broadcasted on the iFrame with a corresponding listener on the payment page waiting to received it (4b). The token is then stored on the payment page’s hidden form and then sent to the backend server.

With the token and private key, the backend server can make API requests (5) to Stripe to charge the customer.
<img src="{{ '/assets/images/blog/PCI_stripe/figure2.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">

The advantage of this solution is that Stripe stores the credit card details and not the website. Stripe complies with PCI (Payment Card Industry Data Security Standards) and must meet high security standards and undertake regular audits. Since Stripe does this on your website behalf, this removes the compliance cost from the website.

Are you locked-in to your selected payment gateway as you no longer have access to your customer’s credit card details? 

No! 

Both Braintree and Stripe allow exporting of credit cards to another PCI compliant payment gateway and the retrieval of any non credit card data via their API.

<img src="{{ '/assets/images/blog/PCI_stripe/figure3.png' | relative_url }}" alt="VSCode" itemprop="image"  class="u-photo">

Note: You still need to prevent malicious code from running on the website that could replicate the credit card form and trick customers into entering their credit card details into this bogus form. This would bypass the payment gateways solution.

### Appendix
- https://stripe.com/docs/security
- https://stripe.com/docs/checkout list of options for checkout
- https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage 
- https://davidwalsh.name/window-postmessage
