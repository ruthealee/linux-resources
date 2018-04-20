### Configuration Cascades
On Linux the default with configuration files is to have configuration that cascades. This means you might have a top level configuration file, and then a number of other configuration files in a subdirectory with specific configuration relating to different aspects of an application or service. Logrotate is a great example of this.

### Includes
Most of the time with config files you'll see them adhere to the practice of having a line that states there's an 'include' this is certainly the case for any OS provided files since most distributions insist on following this convention. Look for a line that states: 'include'.
The convention for additional configuration file locations is usually $main_config_directory/$config_name.d/
For logrotate that looks like: main config file at /etc/logrotate.conf with a line that says 'include /etc/logrotate.d' in which you can put smaller config files. This allows you to override the main settings. The application will parse the main config first, and then the secondary configurations. It uses the values from the main config unless a different value is specified in the secondary configuration. 

### Examples
- /etc/logrotate.conf has an include line that includes configs in logrotate.d
- /etc/nginx/nginx.conf has an include line that includes configs in include /etc/nginx/conf.d/*.conf
