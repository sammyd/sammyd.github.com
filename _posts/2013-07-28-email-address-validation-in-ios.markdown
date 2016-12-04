---
layout: post
title: "Email Address Validation in iOS"
date: 2013-07-28 14:36
comments: true
tags: [iOS, API, cocoapods]
permalink: "blog/:year/:month/:day/:title"
cover: /assets/covers/all_the_way_down.jpg
navigation: true
---

This post is about [GuardPost-ObjectiveC](https://github.com/sammyd/GuardPost-ObjectiveC)
 - an objC wrapper around MailGun's email validation web service. Checkout the
project on [GitHub](https://github.com/sammyd/GuardPost-ObjectiveC) or read on
for more info...

---------

Email address validation can be really hard work. We've all spent many hours
attempting to write a reasonable
[regular expression](http://stackoverflow.com/questions/201323/using-a-regular-expression-to-validate-an-email-address/719543#719543)
to make sure that our users aren't mis-typing their email address, but regex
doesn't catch everything.

MailGun [recently released](http://blog.mailgun.com/post/free-email-validation-api-for-web-forms/)
a new API called GuardPost, which is used for validating
email addresses - checking not only the parts of the email address, but also that
the domain exists and has a responsive mail exchanger. It also offers suggestions
for common mis-spellings of email addresses - e.g. `gmial.com` will result in a
suggestion of `gmail.com`.

This weekend I have built an objC wrapper around the new mailgun GuardPost API,
and I'll explain how to use it here.

<!-- more -->

## GuardPost

GuardPost is part of the web service offered by mailgun, and allows validation
of email addresses. It has a really simple API - details of which can be found
in their [documentation](http://documentation.mailgun.com/api-email-validation.html).
The two methods it offers are as follows:

- **Validation** This takes a string and attempts to determine whether it is a
valid email address. Not only does it check the construction of the string against
a grammar, but also whether the domain exists and whether it supports a mail
exchanger. Suggestions for common mis-spellings are also offered.
- **Parsing** Given a string containing multiple email addresses (comma or
semi-colon delimited) this tool attempts to split them up into valid email addresses
and unparseable string sections.

The GuardPost API is free to use, but requires an API key from mailgun. You can
sign up for a free account [here](https://mailgun.com/signupb?plan=free).


## GuardPost-ObjectiveC

### Installation

GuardPost-ObjectiveC is a friendly wrapper around the mailgun service - based on
[AFNetworking](https://github.com/AFNetworking/AFNetworking). It is packaged up
as a CocoaPod so installation is simple:

{% highlight ruby %}
platform :ios, '5.0'

pod 'GuardPost-ObjectiveC', '~> 0.1.1'
{% endhighlight %}

And then installation (as with every other CocoaPod) is as simple as:

{% highlight bash %}
$ pod install
{% endhighlight %}

This installs the dependencies as well as GuardPost-ObjectiveC itself.


### Usage

The class which provides the required functionality is `GPGuardPost`, and it
is provided by the `GPGuardPost.h` header.

Before attempting to verify an email address you need to specify your mailgun
API key. This is the public API key - with a prefix of `pubkey-`. To set it use
the `setPublicAPIKey:` class method. This only has to be done once per application
so it might make sense to do it in the app delegate:

{% highlight objc %}
#import <GPGuardPost.h>

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // Register the mailgun API Key
    [GPGuardPost setPublicAPIKey:@"pubkey-from-mailgun"];

    // Other app launch options here...

    return YES;
}
...
@end
{% endhighlight %}

Now you're all set to go.

#### Email Address Validation

To verify an email address use the class method
`+validateAddress:success:failure:`. It takes a string for the address, and 2
blocks - one for success, the other for failure.

{% highlight objc %}
[GPGuardPost validateAddress:self.emailField.text
                     success:^(BOOL validity, NSString *suggestion) {
                        NSLog(@"API call successful");
                     }
                     failure:^(NSError *error) {
                        NSLog(@"There was an error: %@", [error localizedDescription]);
                     }];
{% endhighlight %}

The success block has 2 arguments:

- `validity` is a BOOL which specifies whether the email address sent is valid
- `suggestion` is an NSString which has a suggestion for an email address, or is
`nil`. Note that valid email addresses can have a non-`nil` suggestion, and
similarly invalid addresses don't necessarily have a suggestion.

The error block has an `NSError` argument, which will contain details of the
problem - e.g. a `401` message in the event of authorization failure.

#### Email List Parsing

The `+parseListOfAddresses:success:failure` method mirrors the API call provided
by mailgun. Provided a string which contains a list of addresses the method again
has `success` and `failure` callback blocks. The failure block is of the same form
as for email validation.

The success block has 2 `NSArray` arguments - the first for a list of parsed email
addresses, and the second for unparseable parts of the list string. Note that
these email addresses have only been parsed for grammar, and additional validation
can then be performed with calls to the `validateAddress` method.

## Example App

Inside the `GuardPost-ObjectiveC` repo there is a Samples directory, which contains
an example application. This is a really simple app which verifies an email address
as valid and updates the UI as appropriate.

![](/images/2013-07-28-email-validator-app.png)
![](/images/2013-07-28-email-validator-app-invalid.png)


We create the UI in a storyboard and provide the following outlets and methods
in the header file:

{% highlight objc %}
@interface GPViewController : UIViewController <UITextFieldDelegate>

@property (weak, nonatomic) IBOutlet UITextField *emailField;
@property (weak, nonatomic) IBOutlet UILabel *lblValid;
@property (weak, nonatomic) IBOutlet UILabel *lblDidYouMean;
@property (weak, nonatomic) IBOutlet UIButton *btnValidate;
@property (weak, nonatomic) IBOutlet UIActivityIndicatorView *actIndicator;

- (IBAction)btnValidatePressed:(id)sender;
@end
{% endhighlight %}


The implementation which goes alongside this as follows. It's refreshingly
simple:

{% highlight objc %}
#import "GPViewController.h"
#import <GPGuardPost.h>

@implementation ViewController

- (IBAction)btnValidatePressed:(id)sender {
    // Bin off the keyboard
    [self.emailField resignFirstResponder];
    // Start the spinner
    [self.actIndicator startAnimating];
    
    // Send the email address off for validation
    [GPGuardPost validateAddress:self.emailField.text success:^(BOOL validity, NSString *suggestion) {
        // Hide the spinner
        [self.actIndicator stopAnimating];
        
        // Update the validity label
        if(validity) {
            self.lblValid.text = @"VALID";
            self.lblValid.textColor = [UIColor greenColor];
        } else {
            self.lblValid.text = @"INVALID";
            self.lblValid.textColor = [UIColor redColor];
        }
        self.lblValid.hidden = NO;
        
        // And now check for suggestions:
        if(suggestion) {
            self.lblDidYouMean.text = [NSString stringWithFormat:@"Did you mean %@?", suggestion];
            self.lblDidYouMean.hidden = NO;
        }
    } failure:^(NSError *error) {
        // Hide the spinner
        [self.actIndicator stopAnimating];
        self.lblValid.textColor = [UIColor orangeColor];
        self.lblValid.text = @"Error";
        self.lblValid.hidden = NO;
        self.lblDidYouMean.hidden = YES;
        NSLog(@"There was an error: %@", [error localizedDescription]);
    }];
}
@end
{% endhighlight %}

You can see the `success` and `failure` blocks clearly. The majority of this code
is getting the right UI elements to appear at the right time with the correct
content. The call to validate the email address is really simple.

To improve the usability we implement the following `UITextFieldDelegate` method
which will empty the textfield when the keyboard shows, and also hide any previous
result displays:

{% highlight objc %}
#pragma mark - UITextFieldDelegate methods
- (void)textFieldDidBeginEditing:(UITextField *)textField
{
    // Empty the text field
    self.emailField.text = nil;
    // Hide the result fields
    self.lblValid.hidden = YES;
    self.lblDidYouMean.hidden = YES;
}
{% endhighlight %}


## Conclusion

Hopefully you might find this useful. I think it's a great service from mailgun - 
validating email addresses is often a fiddly job, and now it's a lot simpler.

If you have any suggestions/improvements then send a pull request or raise an
issue on the [GitHub](https://github.com/sammyd/GuardPost-ObjectiveC) repo.

If you've enjoyed this post then you should follow me on twitter 
[@iwantmyrealname](https://twitter.com/iwantmyrealname) or adn 
[@samd](https://app.net/samd).

sam