# Automatically mount sharepoint sites and folders in a onedrive client
> :info: **Tldr:** If you have one organization - There is a change. If you have multiple organizations - Don't waste your time - I did it for you.
Really if you have any solution for having automatically mount sharepoint sites across organizations via intune, gpo, registr, powershell, rmm ,... message me.

## The libaryID which does not exist anymore
A lot of tutorials on the web and even microsoft tell you, that you should get a special uri, which you than can deploy via group policy or intune to your clients.
At some point the offical/simple way to get this ur on the sharepoint website has gone :) You could press on the *Sync* button in sharepoint, which showed you some small popup which a little button "Copy LibaryID" and then you was done.

Nowadays it is quite complicated and you could build it manually like this:
`tenantId=<TENANTID>&siteId={<SITEID>}&webId={<WEBID>}&listId={<LISTID>}&webUrl=<SHAREPOINT-SITEURL>&version=1`
Depending on what document on the microsoft or some online blog you need u have to use an url encoding or not.
On my testing it does not matter and you can simple use the unencoded version.

Do encode or decode the url you can simple to this:
Escaping
'[uri]::EscapeDataString($liburl)'
Unescaping
'[uri]::UnescapeDataString($liburl)'

We will call this `$liburl` from now on.
> :warning: There is a well known but undocumented sharepoint/onedrive bug which does now allow this url the be more than 256 characters. Have fun with the encoded version.

## The registry key
Having the `$liburl` does allow you to setup a registry key that will tell the onedrive client to test access this libary and mount it if possible.

1. Create `HKCU:\SOFTWARE\Policies\Microsoft\OneDrive\TenantAutoMount` or `HKLM:\SOFTWARE\Policies\Microsoft\OneDrive\TenantAutoMount`
2. For each libary add a new *REG_SZ* with a name which does not matter and a value with `$liburl`

Example:
`reg add "HKCU:\SOFTWARE\Policies\Microsoft\OneDrive\TenantAutoMount" /v MySharePointLibary /t REG_SZ /d "tenantId=111111-1111-1111-1111-111111111111&siteId={222222-2222-2222-2222-22222222}&webId={333333333-3333-3333-33333-3333333}&listId={333333333-3333-3333-33333-3333333}&webUrl=https://mysharepointuri.sharepoint.com/sites/MySite&version=1" /f `

Building this urls can be painful and i will provide a script to build does soon:
> :info: Hit me up

Even after you have set this key the process can take up to eight-hours - that's what microsoft says.
We can speed this up by setting this:
`reg add "HKCU\Software\Microsoft\OneDrive\Accounts\Business1" /v Timerautomount /t REG_QWORD /d 1 /f `

> :info: Most of the times the sharepoint libary was mounted only after a restart of the device.

> :warning: **This does only work for sharepoint-sites within your orgnanization!** If you have multiple tenants ad have guest-access to other tenants sharepoint-sites. This simple just does not work. No Error, no reason. Just does nothing, even if your user has access to the site!

> :warning: Microsoft says this works only for libaries with max. 5000 files or folders and shared on max 1000 devices. This is a technical microsoft limit.

# The Intune-Way and the gpo
Come on - There you go: [Onedrive - Group Policy](https://docs.microsoft.com/en-us/onedrive/use-group-policy)
You got the `$liburl` - Just paste it in the correct intune policy or office gpo.
If you don't have the gpo you have to install the correct [onedrive admx template](https://docs.microsoft.com/en-us/onedrive/use-group-policy#manage-onedrive-using-group-policy)

# The odopen method
Onedrive does register it on `odopen://` protocol in windows, which we can simply trigger ourselfs with some more complicated urls like this:
`$oduri = "odopen://sync?siteId={222222-2222-2222-2222-22222222}&webId={333333333-3333-3333-33333-3333333}&webUrl=https://mysharepointuri.sharepoint.com/sites/Post&listId={333333333-3333-3333-33333-3333333}&webTitle=<SITETITLE>&listTitle=<LISTTITLE>&onPrem=0&libraryType=3&scope=OPENLIST&userEmail=<USEREMAIL>"`

There are other parameters we stripped for simplicity and they did not chance anything for me :)

The first thing i got a little hiccup was that i have to provide a userEMail. Further more it get's worse in a second. 
What you see there is the what i figured you minimal good and reliable working uri which we can simply throw in the browser or invoke as a process like this:
`Start-Process -filepath [uri]::EscapeDataString($uri)`
> :warning: `$oduri` have to be uri-escaped!

The simplest way to get this uri is:
1. Fire up your browser. 
2. Navigate to the sharepoint site.
3. Open your browser devoloper tools and open the *Network*(Chromium) tab
4. Hit the *sync* button in sharepoint
5. Copy the `adopen://` request from there and strip out all the useless stuff.

## The automatiion
After i gave up on the sharepoint libaryID solution with gpo or intune i went down the new way with the odopen uri.
I have a rmm on all clients or at least i now set this as a requirement from now on that we/you can run some powershell on each client you want to connect to your sharepoint-sites.

> :info: We assume onedrive is already setup on the client. Automatic setup of the onedrive client itself is a hole new rabbit-hole :)
My "small" problem was that i had to deal with the userEmail in the uri which we can simply fetch from registry.
I came up with this little script which i setup on my rmm and run it when i needed on the **user context**:

```Powershell
$uris= @{
  MySharepointSite = "odopen://sync?siteId={222222-2222-2222-2222-22222222}&webId={333333333-3333-3333-33333-3333333}&webUrl=https://mysharepointuri.sharepoint.com/sites/Post&listId={333333333-3333-3333-33333-3333333}&webTitle=<SITETITLE>&listTitle=<LISTTITLE>&onPrem=0&libraryType=3&scope=OPENLIST}

$userMailRegKey="HKCU:\SOFTWARE\Microsoft\OneDrive\Accounts\Business1"
if (-not (Test-Path $userMailRegKey)){
   throw "OneDrive user not found"
}

$mail = (Get-ItemProperty -Path $userMailRegKey -Name UserEMail).UserEMail

$uris.GetEnumerator() | ForEach-Object {
   $uri = [uri]::EscapeDataString($($PSItem.Value + "&userEmail=" + $escapedMail))
   Start-Process -filepath $uri
   Start-Sleep -Seconds 15
}
```

This work's fantastic and saves me a lot of work but there is one small and for me big problem.
This work's only in your organisation with your own sharepoint sites.
I have alot of tenants which work together and shareing there teams and sharepoint websites via guest access.
For whatever reason you can run this `adopen://sync` command from tenant 1 on a user on tenant 2 and the onedrive client does setup the folder locally but not it the same parent folder. 

Example:
User A in Tenant A has permission on a sharepoint site Test on Tenant A and permission on a sharepoint site Test on Tenant B.
When the user A manually goes do each sharepoint sites and run the sync command then the windos file explorer on user A client look like this:

- Onedrive Tenant A
    - My Personal Folders
- Tenant A
    - Test
- Tenant B
    - Test

But if you run it via the odopen uris above it looks like this:
- Onedrive Tenant A
    - My Personal Folders
- Tenant A
    - Test
    - Test(1)

The Folder does get mapped correctly but it just did not made any sense for me, well and still don't.
After some more investigating there is a another uri parameter which seems to be relevant here: the `&userID=<userID>`
If you append the userID in the uri it does work, but at that point i gave up because i could not find any solution how to get the userID locally and i still don't understand why we need it when you already have the users credentials and mail.
Why do we need the userID to setup a folder correctly or why is there not a checkbox on the sharepoint site itself:

[ ] Click this to automatically publish this to Onedrive.

This works, but sucks and we cannot automate:
`odopen://sync?siteId={222222-2222-2222-2222-22222222}&webId={333333333-3333-3333-33333-3333333}&webUrl=https://mysharepointuri.sharepoint.com/sites/Post&listId={333333333-3333-3333-33333-3333333}&webTitle=<SITETITLE>&listTitle=<LISTTITLE>&onPrem=0&libraryType=3&scope=OPENLIST&userId=666666-6666-6666-6666-666666&userEmail=<USERSMAILS>`






