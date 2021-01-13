If you are running a federated authentication with ADFS and your users are coming from outside of your organisation a second factor should be required after successful authentication to get access to Office 365.

But there are situations where you want to skip MFA for a user if a specific authentication mechanism was used. For example if you are logging in with Azure MFA as primary authentication method, immediately getting a Azure MFA as second factor maybe does not make much sense. In this case it maybe ok to skip the second factor.

So if you want to skip MFA while using Azure MFA as the primary authentication method when accessing Office 365 you can do this by modifying the issuance transform rules for Office 365 as follows

Modify default rule
First we have to remove one rule that ships with in default a default configuration (when setup with a recent version of AAD Connect):

@RuleName= "Pass through claim - multifactorauthenticationinstant"
c:[Type == "http://schemas.microsoft.com/ws/2017/04/identity/claims/multifactorauthenticationinstant"]=> issue(claim = c);
The above rules get replaced by the following issuance transformation rules.

Rule: Add custom claims to signal MFA skip
@RuleName = "Add custom claim to signal MFA skip"
c:[Type == "http://schemas.microsoft.com/claims/authnmethodsproviders", Value == "AzurePrimaryAuthentication"]
=> add(Type = "https://treeforest.cloud/skipMFA", Value = "skip"); 
The above rule checks the method used in the process of authentication. In case it was AzurePrimaryAuthentication, we will add a custom claim with the value “skip” for the next rules.

@RuleName = "Add custom no skip claim for other authn methods" 
NOT EXISTS([Type == "https://treeforest.cloud/skipMFA"])
=> add(Type = "https://treeforest.cloud/skipMFA", Value = "no_skip");
The second rule will add our custom claim only if AzurePrimaryAuthentication was NOT used. In this case we will send our custom claim with the value “no_skip”. The order of these two rules is important – so keep an eye on it 😉

Rule: Issue multipleauthn for Azure MFA
If you are using Azure MFA as a primary authentication method, additional MFA can be suppressed with the following claim rule:

@RuleName = "Issue multipleauthn for Azure MFA"
c:[Type == "https://treeforest.cloud/skipMFA", Value == "skip"]
=> issue(Type = "http://schemas.microsoft.com/claims/authnmethodsreferences", Value = "http://schemas.microsoft.com/claims/multipleauthn"); 
If skipMFA with value “skip” is present this rule issues an authnmethodsreference claim with the value multipleauthn. This value is later honored by ADFS access policies and/or Azure AD conditional access policies. So there will be no additional MFA triggered for the user if Azure MFA was used as an primary authentication method.

Rule: Issue multifactorauthenticationinstant for AzureMfa
Just sending a “faked” multipleauthn reference is not enough. You also have to send out an UTC timestamp when the MFA authentication took place. Because we do not have this timestamp as a result of missing MFA we have to use another claim that is related to the authentication. The claim authenticationinstant contains the UTC timestamp for the primary authentication. Issuing this timestamp as multifactorauthenticationsinstant seems practical 😀

@RuleName = "Issue multifactorauthenticationinstant for AzureMfa"
c1:[Type == "https://treeforest.cloud/skipMFA", Value == "skip"] && c2:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/authenticationinstant"]
=> issue(Type = "http://schemas.microsoft.com/ws/2017/04/identity/claims/multifactorauthenticationinstant", Value = c2.Value); 
Rule: Issue multifactorauthenticationinstant clean up
This last rule will issue the default multifactorauthenticationinstant claim if no Azure MFA was used for authentication.

@RuleName= "multifactorauthenticationinstant clean up"
c1:[Type == "https://treeforest.cloud/skipMFA", Value == "no_skip"] && c2:[Type == "http://schemas.microsoft.com/ws/2017/04/identity/claims/multifactorauthenticationinstant"]
=> issue(claim = c2);
Final thoughts
Thats it! You now know how to fake the multipleauthn claim. But be care full – deciding to skip this additional measure can lead to an unsecure identity strategy. But in this special case, if an attacker already has access to the MFA app to provide the first factor, also the second factor shouldn’t be a big problem. So skipping MFA in this case maybe not a big deal from an identity security perspective.
