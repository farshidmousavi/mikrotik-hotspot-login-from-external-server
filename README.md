# mikrotik-hotspot-login-from-external-server
Run MikroTik hotspot, register new users and authenticate customers from external server ( http requests )


In this project we try to
-	Lunch hotspot on a virtual WLAN ( Wireless Network )
-	Customize hotspot login page for access to the internet from external login page using website or IP address
-	Take user information and create user in MikroTik hotspot
-	Check password to authenticate new user

<b>Before start:</b>

You can do <b>step 1</b> to <b>step 7</b> by mikrotik hotspot easy setup, for this go to
```
Winbox > IP > Hotspot > click on add button ( + )
```

<b>Step 1:</b>
Create wireless security profile with no pre-shared key to use in hotspot
```
interface/wireless/security-profiles/add name=hotspot mode=none
```
This code will make a new security profile with no pre-shared key to connect customers without asking Wifi password

<b>Step 2:</b>
Create New WLAN to make an extra wireless for hotspot
```
interface/wireless/add name=wlan2 mtu=1500 arp=enabled mode=ap-bridge ssid=myhotspot master-interface=wlan1 wps-mode=disabled vlan-id=1 security-profile=hotspot disabled=no
```
This code makes a new Virtual WLAN inside your master WLAN
SSID =  myhotspot is your hotspot wireless name, you can choose whatever you want
master-interface = wlan1 is your source to check hotspot users ( I used wireless, You can choose whatever you want )
disabled = no is to run wireless immediately after creation

After run this code you can see the new wireless named in your home/office named myhotspot

<img src="./images/New-WLAN -myhotspot.jpg" align="center">

<b>Step 3:</b>
Create a hotspot IP pool
```
ip/pool/add name=hotspot-ip-pool ranges=10.5.50.2-10.5.50.254
```
This command generates a new range of Ip addresses
We use this range for hotspot connected users

<b>Step 4:</b>
Assign the IP range to wlan2 network
```
ip/address/add address=10.5.50.1/24 network=10.5.50.0 interface=wlan2
```

<b>Step 5:</b>
Create a new hotspot server profile
```
ip/hotspot/profile/add name=hoptspot-profile html-directory=hotspot login-by=http-pap<
```

<b>Step 6:</b>
Add custom URL to static DNS
```
ip/dns/static/add name=myhotspot.my type=A ttl=1d address=10.5.50.1
```
This code makes a static DNS to redirect “myhotspot.my” domain requests to hotspot server IP address

<b>Step 7:</b>
Create Hotspot server and connect it to the new configured WLAN, Ip pool and hotspot profile
```
ip/hotspot/add name=hotspot-server interface=wlan2 address-pool=hotspot-ip-pool profile=hoptspot-profile idle-timeout=00:50:00
```
Now our hotspot created successfully and we can connect to the new SSID ( wireless named )  myhotspot

If you try to connect to this SSID, you will be redirected to the hotspot login page as shown below

<img src="./images/hotspot default login.jpg" align="center">

<b>Step 8:</b>
Customize login page and redirect user to external login page on our server
Now to customize login page we should download and change it.
Go to file list, under hotspot folder right click on hotspot/login.html and download it.


<img src="./images/download file.png" align="center">


Use this simple code to customize login.html page.
```
<!doctype html>
<html lang="en">
<body>
    <form name="redirect" action="http://YOUR_DOMAIN_OR_IP/register.php" method="post">
        <input type="hidden" name="link-login-only" value="$(link-login-only)">
    </form>
    <script langguage="JavaScript">
        document.redirect.submit();
    </script>
</body>
</html>
```


And upload this customized file to your router, in files exactly on old login.html ( you can drag and drop on old file )

<b>Step 9:</b>
Now for redirect user from mikrotik to external page we should add our domain ip/ or server ip to mikrotik   hotspot walled garden IP list.
```
ip/hotspot/walled-garden/add action=allow dst-host=YOUR_DOMAIN_OR_IP
```
This code allow user to open YOUR_DOMAIN_OR_IP without internet access.

<b>Step 10:</b>
Create register.php in your host

```
<?php
$loginLink = $_POST['link-login-only'];
if(!isset($_POST['username'])){
    ?>
    <form action="" method="post">
        <input type="text" name="username"></br>
        <input type="password" name="password"></br>
        <button type="submit">join</button>
    </form>
<?php
}else{
    $router_user = 'Your router username';
    $router_password = 'Your router password';
    $router_ip = 'Your router ip address';
    $ch = curl_init();
    $headers = [
        "Content-Type: application/json",
        "X-Content-Type-Options:nosniff",
        "Accept:application/json",
        "Cache-Control:no-cache"
    ];
    
    curl_setopt($ch, CURLOPT_URL, 'https://'.$router_ip.'/rest/ip/hotspot/user/add');
    curl_setopt($ch, CURLOPT_POSTFIELDS, '{"server":"all","name":$_POST['username'],"password":$_POST['password']}');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($ch, CURLOPT_POST, 1);
    curl_setopt($ch, CURLOPT_USERPWD, $router_user . ':' . $router_password);
    curl_setopt($ch, CURLOPT_PORT, 443);
    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
    curl_setopt($ch,CURLOPT_HTTPHEADER, $headers);

    $result = curl_exec($ch);
    header("Location: ./hotspot.php?login=".$loginLink);
    die();
    if (curl_errno($ch)) {
        echo 'Error:' . curl_error($ch);
    }
    curl_close($ch);
}
?>
```
This page adds the new user in router /ip/hotspot/users after customer submit this form.
You can customize your form and take more info like, email, phone number, full name atc.


<b>Step 11:</b>
Ask the customer to authenticate her/his username and password
```
<?php
if(isset($_GET['login']))
# This scripte wrote by farshid mousavi
# Error lists
# 1 = Connection faild
# 2 = Data not set
# 3 = Add new user faild

?>
<form name="login" action="<?php echo $_GET['login']; ?>" method="post">
    <input type="text" name="username">
    <input type="password" name="password">
<button type="submit">login</button>
</form>
```
This simple page asks for username and password to authenticate customer and give internet access.


<b>Note:If there is a connection error, disable your router's firewall to troubleshoot, you may need to allow connections to port 443 on your router.</b>


