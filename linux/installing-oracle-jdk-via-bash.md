# Installing Oracle JDK via Bash

Oracle's JDK typically requires one to manually accept their license agreement via web interface.  You can download a JDK via `wget` by adding a header that indicates the license agreement has been accepted.

```
export JDK_URL=http://download.oracle.com/otn-pub/java/jdk/8u141-b15/336fa29ff2bb4ef291e347e091f7f4a7/jdk-8u141-linux-x64.rpm
wget --header "Cookie: oraclelicense=accept-securebackup-cookie" $JDK_URL
sudo yum -y localinstall jdk-8u141-linux-x64.rpm
```

The link to the JDK can be obtained by accepting the license agreement via the JDK web page and then copying the link to the proper release.

