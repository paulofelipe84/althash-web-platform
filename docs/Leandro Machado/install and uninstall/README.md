# Steps to make the install and uninstall of Althash Web Platform

**To install**

| Command | Description |
| --- | --- |
| cd /var/www/html | Will enter in the folder of Apache/Nginx |
| sudo git clone https://github.com/machado-rev/althash-web-platform | Will clone this repo |
| cd althash-web-platform | Will enter into the Althash folder |
| sudo mv * /var/www/html  | Will move all files (except the hidden) to the main folder of Apache/Nginx |
| sudo mv /var/www/html/althash-web-platform/.[!.]* /var/www/html/ | Will move all hidden files (except the UNIX folders . and ..) to Apache/Nginx folder |
| cd /var/www/html | Will enter in the folder of Apache/Nginx |
| sudo npm install | Will install the packages |

Just use the script opening the address of your server like: http://yourserver/dist

**To uninstall**

| Command | Description |
| --- | --- |
| cd /var/www/html | Will enter in the folder of Apache/Nginx |
| sudo rm -rf * | Will remove all files (except the hidden) from the main folder of Apache/Nginx |
| sudo rm -rf .* | Will remove all files (except the UNIX folders . and ..) fromthe main folder of Apache/Nginx |
