##利用cx\_freeze在没有Python的Windows上运行Python源代码

在没有安装Python运行环境的Windows机器上运行python程序,大家的通常做法是使用cx_freeze/pyinstaller等打包成exe文件，我也是一直这样做。但是有一个问题，程序做很小的改动的时候，都需要重新打包，然后再拷备过去等一系列复杂的操作。

我发现[goagent项目](https://github.com/goagent/goagent)就是提供的源码，在windows上是用一个`python27.exe`的程序解释源程序。研究了一番，发现goagent是使用py2exe把依赖的库打包成一个zip文件，然后再做一个python27.exe的程序， 由python27.exe来执行，并且有一个单独的项目，名为`pybuild`，项目见[这里](https://github.com/goagent/pybuild)。

我使用[pybuild](https://github.com/goagent/pybuild)的脚本来处理我的程序，希望做成跟goagent一样的效果，结果发现pybuild/py2exe根本搞不定pygtk。

我只有另想办法了，`cx_freeze`打包的时候可以生成一个单独的压缩包，里面包含依赖的库。看来`cx_freeze`可以做成goagent的效果。

我直接使用pybuid里面的[python27.py](https://raw.githubusercontent.com/goagent/pybuild/master/python27.py)来生成python27.exe, 然后再写一个py文件(collect_libs.py)把程序里用到的库全部import一遍。然后利用cx\_freeze把python27.py和collect\_libs.py打包，完成后删除生成的collect\_libs.exe。包含python27.exe的目录就是程序的运行环境， 可以直接使用python27.exe执行python程序。

使用的setup脚本如下


	from cx_Freeze import setup, Executable
	
	# Dependencies are automatically detected, but it might need
	# fine tuning.
	buildOptions = dict(
	        packages = [], excludes = [],
	        include_msvcr = True,
	        include_in_shared_zip = True,
	        )
	
	executables = [
	    Executable('python27.py', 'Console'),
	    Executable('collect_libs.py', "Console")
	]
		
	setup(name='python27',
	      version = '1.0',
	      description = 'the python shell',
	      options = dict(build_exe = buildOptions),
	      executables = executables)



测试了一下，效果还不错。与pybuild/py2exe相比， cx\_freeze把dll,pyd放在目录里面，zip文件里面是pyc文件，而pybuild是把dll, pyd, py文件全放在压缩包里。

需要注意的是，如果cx\_freeze处理使用了gevent的程序，需要对cx\_freeze本身做一些修改， 修改freezer.py文件中的EXTENSION\_LOADER\_SOURCE = 内容中的 `import os, imp, sys`为如下形式,详情参考[这里](https://bitbucket.org/anthony_tuininga/cx_freeze/issue/42/recent-versions-of-gevent-break)


	os = __import__("os")
	imp = __import__("imp")
	sys = __import__("sys")


另外,`cx_freeze`不能处理`egg`文件，需要手动拷备`egg`文件到目标目录下，在程序的最开始加入如下内容，才能正确的import egg文件

	import sys
	import glob
	for egg in glob.glob("*.egg"):
		sys.path.append(egg)
