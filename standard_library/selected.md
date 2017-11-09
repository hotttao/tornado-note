# Tornado 可选的扩展包

可选包:
  - unittest2:是用来在Python 2.6上运行Tornado的测试用例的(更高版本的Python是不需要的)
  - concurrent.futures：
    - 是推荐配合Tornado使用的线程池并且可以支持 tornado.netutil.ThreadedResolver 的用法.
    - 它只在Python 2中被需要，Python 3已经包括了这个标准库.
  - pycurl：
    - 是在 tornado.curl_httpclient 中可选使用的.
    - 需要Libcurl 7.19.3.1 或更高版本;推荐使用7.21.1或更高版本.
  - Twisted：会在 tornado.platform.twisted 中使用.
  - pycares：是一个当线程不适用情况下的非阻塞DNS解决方案.
  - Monotime：
    - 添加对monotonic clock的支持
    - 当环境中的时钟被频繁调整的时候，改善其可靠性. 在Python 3.3中不再需要.
