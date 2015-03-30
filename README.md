# python-Paybox

Simple Python class to help you with the integration of the Paybox payment system

## Requirements

Only the standard library if you intend not to verify the authenticity of the Paybox response via its public key

M2Crypo otherwise (recommended)

    sudo pip install M2Crypto

## Usage

Complete the **settings_example.py** file and rename it **settings.py**

Calling Paybox from a Django view

    from Paybox import Transaction

    transaction = Transaction(
		 PBX_TOTAL=PBX_TOTAL,
		 PBX_PORTEUR=PBX_PORTEUR,
		 PBX_TIME=PBX_TIME,
		 PBX_CMD=PBX_CMD
		)
		
	if production:
	 action = 'https://tpeweb.paybox.com/cgi/MYchoix_pagepaiement.cgi'
	else:
	 action = 'https://preprod-tpeweb.paybox.com/cgi/MYchoix_pagepaiement.cgi'

	form_values = transaction.post_to_paybox()

	return render(request, 'payment.html', {
			'action': action,
			'mandatory': form_values['mandatory'],
			'accessory': form_values['accessory']
		})

How to organise the variables in a template

    <form method="POST" action="{{ action }}">
	<input type="hidden" name="PBX_SITE" value="{{ mandatory.PBX_SITE }}">
	<input type="hidden" name="PBX_RANG" value="{{ mandatory.PBX_RANG }}">
	<input type="hidden" name="PBX_IDENTIFIANT" value="{{ mandatory.PBX_IDENTIFIANT }}">
	<input type="hidden" name="PBX_TOTAL" value="{{ mandatory.PBX_TOTAL }}">
	<input type="hidden" name="PBX_DEVISE" value="{{ mandatory.PBX_DEVISE }}">
	<input type="hidden" name="PBX_CMD" value="{{ mandatory.PBX_CMD }}">
	<input type="hidden" name="PBX_PORTEUR" value="{{ mandatory.PBX_PORTEUR }}">
	<input type="hidden" name="PBX_RETOUR" value="{{ mandatory.PBX_RETOUR }}">
	<input type="hidden" name="PBX_HASH" value="{{ mandatory.PBX_HASH }}">
	<input type="hidden" name="PBX_TIME" value="{{ mandatory.PBX_TIME }}">
	<input type="hidden" name="PBX_HMAC" value="{{ mandatory.hmac }}">
	{% for name, value in accessory.items %}
		{% if value %}
			<input type="hidden" name="{{ name }}" value="{{ value }}">
		{% endif %}
	{% endfor %}
	<input type="submit" value="Proceed to payment">
</form>

Receiving an IPN in a Django view

    from Paybox import Transaction

    transaction = Transaction()
    notification = transaction.verify_notification(response=request.get_full_path(), reference='', total='')

    reference = notification['reference']
    authorization = notification['authorization']
    
    # Paybox Requires a blank 200 response
    return HttpResponse('')

## Understand the Paybox Flow

1. The customer comes to your website and add an item to his basket

2. He creates an account and validates the purchase (or not but you may validate his email address before sending it to Paybox)

3. The purchase is stored in your database, payment set to False (or whatever column you prefer)

4. Your server constructs an html page which redirects the customer to the Paybox website via an hidden form (variables are sent in POST, and you may trigger the redirection automatically via javascript)
	
> post_to_paybox()

> construct_html_form(production=False)

5. Your customer pays, and Paybox send a confirmation to your server. You verify the authenticity of the Paybox callback request, set payment to True and send your customer a thank you email.

To set your IPN url you have to contact customer support. Your ipn url MUST NOT trigger any sort of redirection. Beware of 301 between http:// and http://www.

> verify_notification(response, reference, total, production=False, verify_certificate=True)

> verify_certificate(message, signature)

6. The customer is redirected by the Paybox server on a confirmation page on your server. You may configure this in the Paybox admin. Paybox returns a few variables. Don't do any sort of validation on those urls.

## Paybox installation

### Mandatory variables to be sent by the customer computer with the payment request :

	PBX_SITE (given by Paybox)
	PBX_RANG (Paybox gives you a three digit number, take the last two)
	PBX_IDENTIFIANT (given by Paybox)
	PBX_TOTAL (in cents. 10$ = 1000)
	PBX_DEVISE (number, 978 for €)
	PBX_PORTEUR (email address of the customer)
	PBX_RETOUR (list of variables that paybox will return with the payment confirmation)
	PBX_HASH (How you've keyed-hashed your message. SHA512 in this app)
	PBX_TIME (ISO 8601 Format)
	PBX_HMAC 

### How to send a valid payment call to Paybox

1. Connect to http://preprod-admin.paybox.com to generate a valid secret key. Copy it in the settings of your app or any place SAFE. When in prod, generate a new secret key by going to http://admin.paybox.com

2. In the admin interface, define your redirection urls, and contact the support to set your ipn url

3. Connect everything via your router or anything you're using.

4. In case of trouble, verify that all your mandatory variables are valid. Especially SITE, RANG, IDENTIFIANT. The app constructs a valid redirection form if all your variables are ok.

5. With corrects values for all the variables, the **method post_to_paybox()** constructs a HMAC that ensure the variables are transmitted unaltered to the Paybox server.

## Methods

### post_to_paybox()

returns two dicts :

- mandatory, which contains all the mandatory variables unordered
- accessory, which contains all the other variables you may send to Paybox

### construct_html_form(production=False)

returns a string, which is a valid html form ready to be put inside a template for example

### verify_notification(response, reference, total, production=False, verify_certificate=True)

returns two strings in a dict :

- reference, your reference of the payment
- authorization, the number of authorization that Paybox send on success

### verify_certificate(message, signature)

return True if everything is ok, and horrible errors if the verification fails. This is automatically called by the *verify_notification* method, unless you decide otherwise.
