---
Title: 如何使用 Python 发送邮件
Tags: [类型/译文, 类型/教程, 编程语言/Python, Email]
Original Link: https://www.blog.pythonlibrary.org/2021/09/21/how-to-send-emails-with-python/
Original Title: # How to Send Emails with Python
---

# 如何使用 Python 发送邮件

---

Python 提供了一对非常好的模块，`email` 和 `smtplib`，你可以使用它们去制作电子邮件。
你将会在这里花上一些时间学习如何实际使用这些模块，而不是去探究这俩模块中的各种方法。

具体来说，将涵盖以下内容：

- 电子邮件的基础
- 如何一次性发往多个地址
- 如何在发邮件时使用 TO、CC 和 BCC 行
    > 译注：CC，Carbon copy 抄送；BCC，Blind carbon copy 密送
- 如何使用 `email` 模块添加附件和正文

让我们开始吧！

## 电子邮件基础 - 如何使用 smtplib 发送邮件

`smtplib` 模块使用起来很直观。
你将会看到一个快速示例来说明如何发送电子邮件。

打开你喜欢的 Python IDE 或文本编辑器，然后创建一个新的 Python 文件。
添加以下代码并保存：

```Python
import smtplib

HOST = "mySMTP.server.com"
SUBJECT = "Test email from Python"
TO = "mike@someAddress.org"
FROM = "python@mydomain.com"
text = "Python 3.4 rules them all!"

BODY = "\r\n".join((
        "From: %s" % FROM,
        "To: %s" % TO,
        "Subject: %s" % SUBJECT,
        "",
        text
        ))

server = smtplib.SMTP(HOST)
server.sendmail(FROM, [TO], BODY)
server.quit()
```

这里你只导入了 `smtplib` 模块。
三分之二的代码用于组建电子邮件。
大部分的变量都是不言而喻的，所以你只需关注 `BODY` 那个奇怪的变量。

这里你使用了字符串的 `join()` 方法将多个变量组装成一个字符串，其中每一行都以一个回车（"/r"）和一个换行（"/n"）结束。
如果你把 `BODY` 打印出来，它看上去会像这样：

```Shell
'From: python@mydomain.com\r\nTo: mike@mydomain.com\r\nSubject: Test email from Python\r\n\r\nblah blah blah'
```

之后，你向你的主机的服务器建立了一个连接，然后调用 `smtplib` 模块的 `sendmail` 方法来发送电子邮件。
然后你从服务器断开了连接。
你会注意到这段代码里没有用户名或密码。
如果你的服务要求鉴权，那么你需要加上如下的代码：

```Python
server.login(username, password)
```

你应该在创建服务的实例之后立刻添加它。
通常，你会想把这段代码放进一个函数里，并且搭配一些参数来调用它。
你可能还会想把部分信息放到一个配置文件中。

让我们把这段代码放到函数里吧。

```Python
import smtplib

def send_email(host, subject, to_addr, from_addr, body_text):
    """
    Send an email
    """
    BODY = "\r\n".join((
            "From: %s" % from_addr,
            "To: %S" % to_addr,
            "Subject: %s" % subject,
            "",
            body_text
            ))
    server = smtplib.SMTP(host)
    server.sendmail(from_addr, [to_addr], BODY)
    server.quit()

if __name__ == "__main__":
    host = "mySMTP.server.com"
    subject = "Test email from Python"
    to_addr = "mike@someAddress.org"
    from_addr = "python@mydomain.com"
    body_text = "Python rules them all!"
    send_email(host, subject, to_addr, from_addr, body_text)
```

现在你可以观察这个函数本身来了解这段代码实际上有多小。
13 行！
并且如果不把 `BODY` 中的每一项都放在它们现在的那几行里，你还可以进一步缩短它；不过可读性或许不如现在。
现在你将添加一个配置文件来保存服务器信息和发件人地址。

为什么要这么做呢？
许多组织使用不同的邮件服务器来发送邮件，或者如果邮件服务器更新了并且名字也发生了改变，那么你只需要变更配置文件而不是代码。
如果你的公司被收购合并了，同样的方法也可以应用在发件人地址上。

让我们看看配置文件（保存为 `email.ini`）：

```ini
[smtp]
setver = some.server.com
from_addr = python@mydomain.com
```

这是个很简单的配置文件。
文件内有一个章节被标记为 `smtp`，其中又有两项：`server` 和 `from_addr`。
你将会使用 `ConfigParser` 来读取这个文件，并将其转化为一个 Python 字典。
这是这段代码更新后的版本（保存为 `smtp_config.py`）：

```Python
import os
import smtplib
import sys

from configparser import ConfigParser

def send_email(subject, to_addr, body_text):
    """
    Send an email
    """
    base_path = os.path.dirname(os.path.abspath(__file__))
    config_path = os.path.join(base_path, "email.ini")

    if os.path.exists(config_path):
        cfg = ConfigParser()
        cfg.read(config_path)
    else:
        print("Config not found! Exiting!")
        sys.exit(1)

    host = cfg.get("smtp", "server")
    from_addr = cfg.get("smtp", "from_addr")

    BODY = "\r\n".join((
            "From: %s" % from_addr,
            "To: %s" % to_addr,
            "Subject: %s" % subject,
            "",
            body_text
            ))
    server = smtplib.SMTP(host)
    server.sendmail(from_addr, [to_addr], BODY)
    server.quit()

if __name__ == "__main__":
    subject = "Test email from Python"
    to_addr = "mike@someAddress.org"
    body_text = "Python rules them all!"
    send_email(subject, to_addr, body_text)
```

你往这段代码里加入了一点检查。
你首先抓取这个脚本本身的路径，由 `base_path` 所代表。
接着，将该路径和文件名组合，以获取一个指向配置文件的完整的合格路径。
然后你检查该文件是否存在。

如果路径存在，创建一个 `ConfigParser`；如果不在，打印一条信息并退出脚本。
为了安全起见，你应该在调用 `ConfigParser.read()` 的前后加上异常处理，因为可能文件存在但已经损坏或没有权限打开它，这都将引发一个异常。

这会是个小项目，你可以自己去尝试。
总之，假设一切都顺利，`ConfigParser` 也就成功创建了。
现在你可以用常规的 `ConfigParser` 语法提取 `host` 和 `from_addr` 信息。

现在你已经准备好去学习如何同时发送多封邮件！

## 一次性发送多封邮件

能够一次性发送多个邮件是个很好的功能。

继续稍稍修改最后一个例子，以使你可以发送多封邮件！

```Python
import os
import smtplib
import sys

from configparser import ConfigParser

def send_email(subject, body_text, emails):
    """
    Send an email
    """
    base_path = os.path.dirname(os.path.abspath(__file__))
    config_path = os.paht.join(base_path, "email.ini")

    if os.path.exists(config_path):
        cfg = ConfigParser()
        cfg.read(config_path)
    else:
        print("Config not found! Exiting!")
        sys.exit(1)

    host = cfg.get("smtp", "server")
    from_addr = cfg.get("smtp", "from_addr")

    BODY = "\r\n".join((
            "From: %s" % from_addr,
            "To: %s" % ', '.join(emails),
            "Subject: %s" % subject,
            "",
            body_text
            ))
    server = smtplib.SMTP(host)
    server.sendmail(from_addr, emails, BODY)
    server.quit()

if __name__ == "__main__":
    emails = ["mike@someAddress.org", "someone@someAddress.org"]
    subject = "Test email from Python"
    body_text = "Python rules them all!"
    send_email(subject, body_text, emails)
```

注意到在这个例子里，你移除了 `to_addr` 参数并且添加了一个 `emails` 参数，它是一个邮件地址的列表。
要做到这一点，你需要在 `BODY` 的 `To:` 部分创建一个以逗号分隔的字符串，并将这个邮件列表传递给 `sendmail` 方法。
因此，你执行了以下操作来创建一个简单的逗号分隔的字符串：`', '.join(emails)`。
简单，对吧？

## 发邮件时使用 TO、CC 和 BCC 行

现在你只需要弄清楚如何使用 CC 和 BCC 字段。

让我们创建一个支持这个功能的新版代码。

```Python
import os
import smtplib
import sys

from configparser import ConfigParser

def send_email(subject, body_text, to_emails, cc_emails, bcc_emails):
    """
    Send an email
    """
    base_path = os.path.dirname(os.path.abspath(__file__))
    config_path = os.path.join(base_path, "email.ini")

    if os.path.exists(config_path):
        cfg = ConfigParser()
        cfg.read(config_path)
    else:
        print("Config not found! Exiting!")
        sys.exit(1)

    host = cfg.get("smtp", "server")
    from_addr = cfg.get("smtp", "from_addr")

    BODY = "\r\n".join((
            "From: %s" % from_addr,
            "To: %s" % ', '.join(to_emails),
            "CC: %s" % ', '.join(cc_emails),
            "BCC: %s" % ', '.join(bcc_emails),
            "Subject: %s" % subject,
            "",
            body_text
            ))
    emails = to_emails + cc_emails + bcc_emails

    server = smtplib.SMTP(host)
    server.sendmail(from_addr, emails, BODY)
    server.quit()

if __name__ == "__main__":
    emails = ["mike@somewhere.org"]
    cc_emails = ["someone@gmail.com"]
    bcc_emails = ["schmuck@newtel.net"]

    subject = "Test email from Python"
    body_text = "Python rules them all!"
    send_email(subject, body_text, emails, cc_emails, bcc_emails)
```

这段代码中，你传递了三个列表，每个列表都有一个邮件地址。
一如此前，你创建了 **CC** 和 **BCC** 字段，但你仍然需要合并三个列表并将合并好的列表传递给 `sendmail()` 方法。

在诸如 StackOverflow 这样的论坛上有些讨论，一些邮件客户端会以一种奇怪的方式处理 BCC 字段，以使收件人可以通过邮件头看到密送列表。
我没能确认这一行为，但我知道 Gmail 能成功地从邮件头部剥离 BCC 信息。

现在你已准备好继续使用 Python 的 `email` 模块了。

## 使用邮件模块添加附件和正文

现在你要带着之前章节学到的知识，结合 Python 的 `email` 模块，来发送附件。

`email` 模块使添加附件变得非常简单。
代码如下：

```Python
import os
import smtplib
import sys

from configparser import ConfigParser
from email import encoders
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email.mime.multipart import MIMEMultipart
from email.utils import formatdate

def sned_email_with_attachment(subject, body_text, to_emails,
                               cc_emails, bcc_emails, file_to_attach):
    """
    Send an email with an attachment
    """
    base_path = os.path.dirname(os.path.abspath(__file__))
    config_path = os.path.join(base_path, "email.ini")
    header = 'Content-Disposition', 'attachment; filename="%s"' % file_to_attach

    # get the config
    if os.path.exists(config_path):
        cfg = ConfigParser()
        cfg.read(config_path)
    else:
        print("Config not found! Exiting!")
        sys.exit(1)

    # extract server and from_addr from config
    host = cfg.get("smtp", "server")
    from_addr = cfg.get("smtp", "from_addr")

    # create the message
    msg = MIMEMultipart()
    msg["From"] = from_addr
    msg["Subject"] = subject
    msg["Date"] = formatdate(localtime=True)
    if body_text:
        msg.attach( MIMEText(body_text) )

    msg["To"] = ', '.join(to_emails)
    msg["cc"] = ', '.join(cc_emails)

    attachment = MIMEBase('application', "octet-stream")
    try:
        with open(file_to_attach, "rb") as fh:
            data = fh.read()
        attachment.set_payload(data)
        encoders.encode_base64(attachment)
        attachment.add_header(*header)
        msg.attach(attachment)
    except IOError:
        msg = "Error opening attachment file %s" % file_to_attach
        print(msg)
        sys.exit(1)

    emails = to_emails + cc_emails

    server = smtplib.SMTP(host)
    server.sendmail(from_addr, emails, msg.asString())
    server.quit()

if __name__ == "__main__":
    emails = ["mike@someAddress.org", "nedry@jp.net"]
    cc_emails = ["someone@gmail.com"]
    bcc_emails = ["anonymous@circe.org"]

    subject = "Test email with attachment from Python"
    body_text = "This email contains an attachment!"
    path = "/path/to/some/file"
    send_email_with_attachment(subject, body_text, emails,
                               cc_emails, bcc_emails, path)
```

这里，你重新命名了你的函数，并添加了一个新参数 `file_to_attach`。
你还需要添加一个头部，并创建一个 `MIMEMultipart` 对象。
该头部可以在你添加附件前的任意时间点创建。

你向该 `MIMEMultipart` 对象（`msg`）添加元素，就像向字典中添加键一样。
你会注意到，你必须使用 `email` 模块的 `formatdate` 方法去插入合适格式的数据。

要添加消息正文，你需要创建一个 `MIMEText` 实例。
如果你留意了，你会发现你没有添加密送信息，但你可以遵循上述代码的例子轻松添加。

接着添加附件。
你把它包在了异常处理中，并使用 `with` 语句提取文件且放在了 `MIMEBase` 对象中。
最后，你将它添加到了 `msg` 变量中并发送出去。
注意，你得把 `msg` 转变成字符串才能用在 `sendmail()` 方法里。

## 总结

现在你了解了如何使用 Python 发送邮件。
对于那些喜欢小项目的人来说，应该倒回去在 `server.sendmail` 部分的代码附近添加额外的异常处理，以防在此过程中发生一些奇怪的情况。

一个例子是 `SMTPAuthenticationError` 或 `SMTPConnectError`。
你也可以在文件的附件处加强错误处理，以捕捉其他错误。
最后，你可能想获取这些不同的邮件列表并创建一个删除重复项的规范化列表。
如果你正从文件中读取邮件地址列表，这一点尤其重要。

此外，注意你的发信地址是假的。
你可以用 Python 和其他编程语言伪装邮件，但这是糟糕的礼节，基于你的居住地，它还可能是非法的。
警告过你了！

明智地使用你的知识，享受 Python 带来的乐趣和益处。