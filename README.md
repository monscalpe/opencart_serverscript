# Opencart Serverside Script - Overview

(Important info - the following has been tested with ubuntu 14.04, using the "ubuntu" user. The contents of this repo was put into a folder called "scripts" and placed in the /home/ubuntu/ directory. Please bear this in mind so that you can make the relevant changes to the scripts.)


This repository contains a number of scripts which complete the following:

1) A cron job runs the newcustomercheck script, which will query the opencartdb in MySQL for a list of COMPLETE transactions / product purchases

2) Newcustomercheck script contains commands which compares the results of this new query to an old query, by putting the results of the MySQL query in a file called "newquery", and compares the contents to a file called "oldquery" 

3) Newcustomercheck script contains command which extracts any differences between the new query and the old query (which are new orders) and creates a "neworders" file

4) Newcustomerchecl runs a separate script called "creator", which runs against the "neworders" file. This script will extract the relevant information and push these as variables into the var folder. This script will run each line one by one for each new customer.

5) For every new customer, a new script is called depending on the product id - in this case I have assumed a product with id "50", so I call upon a script in the productscripts folder called "50"

6) The product script will run (in this case "50", which simply sends a html email with a zipped attachment using sendmail)

___________________

# Newcustomercheck script

These are contents of newcustomercheck script

##sourcing the variable username and password for mysql query - make sure you complete this prior to running the script or it WILL fail - you could include the variables in the newcustomercheck file, however I chose to leave it in an external file and simply source it to here.

source /home/ubuntu/scripts/var/newcustomervar

##making the MySQL query, which selects firstname, lastname, product_id for any order which is in complete status (5), and sends output to a file called newquery

mysql -u $username --password=$password -e 'select concat(firstname, " ", lastname) as 'name', email, product_id from opencartdb.oc_order, opencartdb.oc_order_product where order_status_id= 5 and oc_order.order_id = oc_order_product.order_product_id;' > /home/ubuntu/scripts/newquery

##this removes the first line of the file, which would contain the column headers, outputs it to a tmpfile, then mvs the temp file back and renames to original which overwrites it

sed '1d' /home/ubuntu/scripts/newquery > /home/ubuntu/scripts/tmpfile; mv /home/ubuntu/scripts/tmpfile /home/ubuntu/scripts/newquery

##this command compares the contents of the newquery file to that of the oldquery file. Any differences are output to a file called neworders

comm -2 -3 <(sort /home/ubuntu/scripts/newquery) <(sort /home/ubuntu/scripts/oldquery) > /home/ubuntu/scripts/neworders

##the next command calls up the creator script, running it against the neworders file as an input

/home/ubuntu/scripts/creator /home/ubuntu/scripts/neworders

##this command then copies the newquery file to the oldquery file - it is the last thing which runs as a whole once all the other scripts have been called and completed.

cp /home/ubuntu/scripts/newquery /home/ubuntu/scripts/oldquery

___________________

# Creator script

## The following command reads each line one by one in the neworders file, breaking down the output to new variables. It also then exports these variables to the relevant files in the var folder. Once exported, it will then run any individual product script for a given product it finds in the productscripts folder. It will complete this process for every line until there is no line.

while IFS='' read -u10 -r line || [[ -n "$line" ]]; do
	firstname=$(echo $line | awk '{print $1}')
	surname=$(echo $line | awk '{print $2}')
	email=$(echo $line | awk '{print $3}')
	product=$(echo $line | awk '{print $4}')
	echo firstname=\"$firstname\" > /home/ubuntu/scripts/var/firstname
	echo surname=\"$surname\" > /home/ubuntu/scripts/var/surname
	echo email=\"$email\" > /home/ubuntu/scripts/var/email
	echo product=\"$product\" > /home/ubuntu/scripts/var/product
	/home/ubuntu/scripts/productscripts/$product
done 10< "$1"

___________________

# Product Script (example here is "50", which sends HTML email with attachment using sendmail)

##this script calls the external variables established in previous script, and uses them in this one to attach a given file to a html email, which we then send using sendmail.

source /home/ubuntu/scripts/var/firstname
source /home/ubuntu/scripts/var/surname
source /home/ubuntu/scripts/var/email
source /home/ubuntu/scripts/var/product
UN="$(sudo cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 6 | head -n 1)"
PW="$(sudo cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 8 | head -n 1)"
DATE="$(date +%Y/%m/%d\ %H:%M:%S)"

##this is where we compile and send the email

from="hello@yourwebsite.com"
subject="yoursubject"
boundary=`uuidgen`
declare -a attachments
attachments=( "/home/ubuntu/somezipfile.zip" )

## Build headers
{

printf '%s\n' "From: $from
To: $email
Subject: $subject
Date: $DATE
Mime-Version: 1.0
Content-Type: multipart/mixed; boundary=\"$boundary\"

--${boundary}
Content-Type: text/html;charset=UTF-8
Content-Transfer-Encoding: quoted-printable
Content-ID: html-body

<!DOCTYPE html>
<html>
<head><title></title></head>
<body>
<p>Dear <b>$firstname $surname</b>,</p><br>
<p>This email contains blah blah blah</p>
<p>Your credentials are attached to this email and can be used <b>straight away</b>.</p>
<p>Thank you for shopping at yourshop</p>
<hr>
<footer>
<a href="https://www.yoursite.com"><img src="https://www.yourimage.com/image.png"></a>
</footer>
</body>
</html>"

## print last boundary with closing --
printf '%s\n' "--${boundary}--"

} | sendmail -t -oi -f "hello@yoursite.com"
