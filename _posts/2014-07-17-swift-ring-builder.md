---
layout: post
title: "opensatck swift-ring-builder源码分析"
description: ""
category: 
tags: []
---
{% include JB/setup %}

swift中关键的部分便是ring，那ring是如何创建的呢？

我们来看bin目录下的swift-ring-builder:
{% highlight py linenos %}
import sys

from swift.cli.ringbuilder import main


if __name__ == "__main__":
    sys.exit(main())
{% endhighlight %}
调用swift.cli.ringbuilder中的main函数:
{% highlight py lineos %}
def main(arguments=None):
    global argv, backup_dir, builder, builder_file, ring_file
    if arguments:
        argv = arguments
    else:
        #from sys import argv as sys_argv
        argv = sys_argv

    #参数只有一个时，也就是只是swift-ring-builder时，打印帮助信息
    if len(argv) < 2:
        print "swift-ring-builder %(MAJOR_VERSION)s.%(MINOR_VERSION)s\n" % \
              globals()
        print Commands.default.__doc__.strip()
        print
        #去掉内置方法和default方法
        cmds = [c for c, f in Commands.__dict__.iteritems()
                if f.__doc__ and c[0] != '_' and c != 'default']
        cmds.sort()
        for cmd in cmds:
            #打印方法的doc
            print Commands.__dict__[cmd].__doc__.strip()
            print
        print parse_search_value.__doc__.strip()
        print
        #wrap用来包装一段文本，返回值是文本每一行的列表
        for line in wrap(' '.join(cmds), 79, initial_indent='Quick list: ',
                         subsequent_indent='            '):
            print line
        print('Exit codes: 0 = operation successful\n'
              '            1 = operation completed with warnings\n'
              '            2 = error')
        exit(EXIT_SUCCESS)

    #parse_builder_ring_filename_args函数从第二个参数中解析出builder_file和ring_file
    builder_file, ring_file = parse_builder_ring_filename_args(argv)

    #如果builder_file已存在，加载这个file
    if exists(builder_file):
        builder = RingBuilder.load(builder_file)
    elif len(argv) < 3 or argv[2] not in('create', 'write_builder'):
        print 'Ring Builder file does not exist: %s' % argv[1]
        exit(EXIT_ERROR)

    #创建备份目录
    backup_dir = pathjoin(dirname(argv[1]), 'backups')
    try:
        mkdir(backup_dir)
    except OSError as err:
        if err.errno != EEXIST:
            raise

    if len(argv) == 2:
        command = "default"
    else:
        command = argv[2]
    if argv[0].endswith('-safe'):
        try:
            with lock_parent_directory(abspath(argv[1]), 15):
                Commands.__dict__.get(command, Commands.unknown.im_func)()
        except exceptions.LockTimeout:
            print "Ring/builder dir currently locked."
            exit(2)
    else:
        #执行command对应的方法
        Commands.__dict__.get(command, Commands.unknown.im_func)()
{% endhighlight %}
我们来看下Commands里有多少方法：
create， default， search， list_parts， add， set_weight， set_info， remove， rebalance， validate， write\_ring， write\_builder， set\_min\_part\_hours， set\_replicas

通过建环的过程来分析每一个函数。
1.swift-ring-builder account.builder create 18 3 1
create 方法：






























