---
title: IO-File
date: 2016-9-9 10:46:55
---

java 中关于文件操作的类

## java.io.File

> An abstract representation of file and directory pathnames.

File类是 文件或者目录 的路径名称（pathname）的抽象。

``` java
public class File implements Serializable, Comparable<File>
{
    /**
     * This abstract pathname's normalized pathname string.  A normalized
     * pathname string uses the default name-separator character and does not
     * contain any duplicate or redundant separators.
     */
    private String path;

    /**
     * The length of this abstract pathname's prefix, or zero if it has no
     * prefix.
     */
    private transient int prefixLength;

	// ...
}
```

path 和 prefixLength 是 File 类中惟一的两个实例变量，所以创建一个File对象，其实就是这两个变量，反之，这两个变量也就是 File 对象所提供的抽象。可以看到 File 对象其实就是一个路径字符串。

### 路径名

File 类将 pathname 抽象为两部分：

1. An optional system-dependent prefix string, such as a disk-drive specifier, "/" for the UNIX root directory, or "\\\\" for a Microsoft Windows UNC pathname

2. A sequence of zero or more string names

pathname 开头以：平台相关的前缀名，例如： "c:", "/", "\\\\" 等等。然后是一系列目录名和文件名。

绝对路径和相对路径：

* 绝对路径

	绝对路径，包含了可以直接定位到文件位置的完整路径。

* 相对路径

	相对路径，则是相对于一个默认的 work directory. 默认情况下，io包里的类，将把相对路径解析成相对于 current user directory 的路径。而 current user directory 则是由 系统属性 user.dir 来设置的。通常情况下这个路径是 JVM 被调用的那个目录。

## JDK实现不同平台的文件系统的抽象

### 编译 JDK 就引用不同的实现

下面的代码摘自： openjdk\jdk\make\java\java\Makefile

``` makefile
ifeq ($(PLATFORM),windows)
FILES_java += 	java/io/Win32FileSystem.java \
		java/io/WinNTFileSystem.java \
		java/util/prefs/WindowsPreferences.java \
                java/util/prefs/WindowsPreferencesFactory.java

FILES_c    +=   ProcessImpl_md.c \
		Win32FileSystem_md.c \
		WinNTFileSystem_md.c \
		canonicalize_md.c \
		dirent_md.c \
		TimeZone.c \
		TimeZone_md.c \
		WindowsPreferences.c \
		sun/security/provider/WinCAPISeedGenerator.c \
		sun/io/Win32ErrorMode.c

else # PLATFORM
FILES_java += 	java/lang/UNIXProcess.java \
		java/io/UnixFileSystem.java \
		java/util/prefs/FileSystemPreferences.java \
                java/util/prefs/FileSystemPreferencesFactory.java \

FILES_c    +=   UNIXProcess_md.c \
		UnixFileSystem_md.c \
		canonicalize_md.c \
		TimeZone.c \
		TimeZone_md.c \
		FileSystemPreferences.c

INIT += $(GENSRCDIR)/java/lang/UNIXProcess.java

endif # PLATFORM
```

可以当 JDK 在编译到不同的平台的时候，会依据目标平台的，提供不同的实现，例如：FileSystem 的实现，在 windows 平台上是：Win32FileSystem.java, WinNTFileSystem.java, Win32FileSystem_md.c, WinNTFileSystem_md.c 来实现，而在 unix 平台是则是由：UnixFileSystem.java, UnixFileSystem_md.c 来提供实现。

``` c
// windows 平台的实现
// 摘自：openjdk\jdk\src\windows\native\java\io\FileSystem_md.c
JNIEXPORT jobject JNICALL
Java_java_io_FileSystem_getFileSystem(JNIEnv *env, jclass ignored)
{
    initializeWindowsVersion();
    if (onNT) {
        return JNU_NewObjectByName(env, "java/io/WinNTFileSystem", "()V");
    } else {
        return JNU_NewObjectByName(env, "java/io/Win32FileSystem", "()V");
    }
}

// unix 平台的实现
// 摘自：openjdk\jdk\src\solaris\native\java\io\FileSystem_md.c
JNIEXPORT jobject JNICALL
Java_java_io_FileSystem_getFileSystem(JNIEnv *env, jclass ignored)
{
    return JNU_NewObjectByName(env, "java/io/UnixFileSystem", "()V");
}
```

Java 中的 File 对象获得文件系统的抽象	

``` java
// java.io.FileSystem
abstract class FileSystem {
    /**
     * Return the FileSystem object representing this platform's local
     * filesystem.
     */
    public static native FileSystem getFileSystem();

	// ...
}

public class File implements Serializable, Comparable<File>
{
    /**
     * The FileSystem object representing the platform's local file system.
     */
    static private FileSystem fs = FileSystem.getFileSystem();

	// ...
}
```

File 类通过 FileSystem.getFileSystem 方法获得JVM运行时的本地文件系统。所有和平台相关的方法，都将转发给 fs（FileSystem） 这个对象。而这个对象的实现则是由不同平台的子类来实现的。

## UnixFileSystem

UnixFileSystem 是 java.io.FileSystem 在 unix 平台上的实现。

### 文件权限检测：

* canExecute
* canRead
* canWrite

使用 fs.checkAccess 来实现。

``` c
JNIEXPORT jboolean JNICALL
Java_java_io_UnixFileSystem_checkAccess(JNIEnv *env, jobject this,
                                        jobject file, jint a)
{
    jboolean rv = JNI_FALSE;
    int mode = 0;
    switch (a) {
    case java_io_FileSystem_ACCESS_READ:
        mode = R_OK;
        break;
    case java_io_FileSystem_ACCESS_WRITE:
        mode = W_OK;
        break;
    case java_io_FileSystem_ACCESS_EXECUTE:
        mode = X_OK;
        break;
    default: assert(0);
    }
    WITH_FIELD_PLATFORM_STRING(env, file, ids.path, path) {
		// access 是 unix 平台的对文件权限检测的API
        if (access(path, mode) == 0) {
            rv = JNI_TRUE;
        }
    } END_PLATFORM_STRING(env, path);
    return rv;
}
```

>  access() checks whether the calling process can access the file pathname.  If pathname is a symbolic link, it is deref-erenced.

### 创建文件和目录

* createNewFile

	由 fs.createFileExclusively(path) 来实现

* mkdir

	由 fs.createDirectory(this) 来实现。

``` c
JNIEXPORT jboolean JNICALL
Java_java_io_UnixFileSystem_createFileExclusively(JNIEnv *env, jclass cls,
                                                  jstring pathname)
{
    jboolean rv = JNI_FALSE;

    WITH_PLATFORM_STRING(env, pathname, path) {
        int fd;
        if (!strcmp (path, "/")) {
            fd = JVM_EEXIST;    /* The root directory always exists */
        } else {
            fd = JVM_Open(path, JVM_O_RDWR | JVM_O_CREAT | JVM_O_EXCL, 0666);
        }
        if (fd < 0) {
            if (fd != JVM_EEXIST) {
                JNU_ThrowIOExceptionWithLastError(env, path);
            }
        } else {
            JVM_Close(fd);
            rv = JNI_TRUE;
        }
    } END_PLATFORM_STRING(env, path);
    return rv;
}

JNIEXPORT jboolean JNICALL
Java_java_io_UnixFileSystem_createDirectory(JNIEnv *env, jobject this,
                                            jobject file)
{
    jboolean rv = JNI_FALSE;

    WITH_FIELD_PLATFORM_STRING(env, file, ids.path, path) {
	// mkdir() attempts to create a directory named pathname.
	// The  argument  mode specifies the permissions to use.
        if (mkdir(path, 0777) == 0) {
            rv = JNI_TRUE;
        }
    } END_PLATFORM_STRING(env, path);
    return rv;
}
```

## WinNTFileSystem

### 在 windows 平台上实现的 checkAccess

``` c
JNIEXPORT jboolean
JNICALL Java_java_io_WinNTFileSystem_checkAccess(JNIEnv *env, jobject this,
                                                 jobject file, jint access)
{
    DWORD attr;
    WCHAR *pathbuf = fileToNTPath(env, file, ids.path);
    if (pathbuf == NULL)
        return JNI_FALSE;
    attr = GetFileAttributesW(pathbuf);
    attr = getFinalAttributesIfReparsePoint(pathbuf, attr);
    free(pathbuf);
    if (attr == INVALID_FILE_ATTRIBUTES)
        return JNI_FALSE;
    switch (access) {
    case java_io_FileSystem_ACCESS_READ:
    case java_io_FileSystem_ACCESS_EXECUTE:
        return JNI_TRUE;
    case java_io_FileSystem_ACCESS_WRITE:
        /* Read-only attribute ignored on directories */
        if ((attr & FILE_ATTRIBUTE_DIRECTORY) ||
            (attr & FILE_ATTRIBUTE_READONLY) == 0)
            return JNI_TRUE;
        else
            return JNI_FALSE;
    default:
        assert(0);
        return JNI_FALSE;
    }
}
```

主要是调用 GetFileAttributesW 这个API来获得文件的属性信息：
> The GetFileAttributes function retrieves a set of FAT file system attributes for a specified file or directory.

### 创建文件和目录

``` c
JNIEXPORT jboolean JNICALL
Java_java_io_WinNTFileSystem_createFileExclusively(JNIEnv *env, jclass cls,
                                                   jstring path)
{
    HANDLE h = NULL;
    WCHAR *pathbuf = pathToNTPath(env, path, JNI_FALSE);
    if (pathbuf == NULL)
        return JNI_FALSE;
	// The CreateFile function creates or opens a file, file stream, 
 	// directory, physical disk, volume, console buffer, tape drive, 
	// communications resource, mailslot, or named pipe. 
    h = CreateFileW(
        pathbuf,                              /* Wide char path name */
        GENERIC_READ | GENERIC_WRITE,         /* Read and write permission */
        FILE_SHARE_READ | FILE_SHARE_WRITE,   /* File sharing flags */
        NULL,                                 /* Security attributes */
        CREATE_NEW,                           /* creation disposition */
        FILE_ATTRIBUTE_NORMAL |
            FILE_FLAG_OPEN_REPARSE_POINT,     /* flags and attributes */
        NULL);

    if (h == INVALID_HANDLE_VALUE) {
        DWORD error = GetLastError();
        if ((error != ERROR_FILE_EXISTS) && (error != ERROR_ALREADY_EXISTS)) {
            // return false rather than throwing an exception when there is
            // an existing file.
            DWORD a = GetFileAttributesW(pathbuf);
            if (a == INVALID_FILE_ATTRIBUTES) {
                SetLastError(error);
                JNU_ThrowIOExceptionWithLastError(env, "Could not open file");
            }
         }
         free(pathbuf);
         return JNI_FALSE;
        }
    free(pathbuf);
    CloseHandle(h);
    return JNI_TRUE;
}

JNIEXPORT jboolean JNICALL
Java_java_io_WinNTFileSystem_createDirectory(JNIEnv *env, jobject this,
                                             jobject file)
{
    BOOL h = FALSE;
    WCHAR *pathbuf = fileToNTPath(env, file, ids.path);
    if (pathbuf == NULL) {
        /* Exception is pending */
        return JNI_FALSE;
    }
	// The CreateDirectory function creates a new directory.
    h = CreateDirectoryW(pathbuf, NULL);
    free(pathbuf);

    if (h == 0) {
        return JNI_FALSE;
    }

    return JNI_TRUE;
}
```

其它的方法实现都类似，就是使用不同平台的 文件系统 的API 来实现 FileSystem 的功能。


## java.nio.files

> Defines interfaces and classes for the Java virtual machine to access files, file attributes, and file systems.

## java.nio.file.FileSystems.getDefault

``` java
// openjdk\jdk\src\solaris\classes\sun\nio\fs\DefaultFileSystemProvider.java
/**
 * Returns the default FileSystemProvider.
 */
public static FileSystemProvider create() {
    String osname = AccessController
        .doPrivileged(new GetPropertyAction("os.name"));
    if (osname.equals("SunOS"))
        return createProvider("sun.nio.fs.SolarisFileSystemProvider");
    if (osname.equals("Linux"))
        return createProvider("sun.nio.fs.LinuxFileSystemProvider");
    throw new AssertionError("Platform not recognized");
}

// openjdk\jdk\src\windows\classes\sun\nio\fs\DefaultFileSystemProvider.java
/**
 * Creates default provider on Windows
 */
public class DefaultFileSystemProvider {
    private DefaultFileSystemProvider() { }
    public static FileSystemProvider create() {
        return new WindowsFileSystemProvider();
    }
}

// FileSystems 类获得 defaultFileSystem
// returns default file system
private static FileSystem () {
    // load default provider
    FileSystemProvider provider = AccessController
        .doPrivileged(new PrivilegedAction<FileSystemProvider>() {
            public FileSystemProvider run() {
                return getDefaultProvider();
            }
        });

    // return file system
	// 最终调用 provider.getFileSystem 来获得文件系统。
    return provider.getFileSystem(URI.create("file:///"));
}

public class WindowsFileSystemProvider
    extends AbstractFileSystemProvider
{
    private static final String USER_DIR = "user.dir";
    private final WindowsFileSystem theFileSystem;

    public WindowsFileSystemProvider() {
        theFileSystem = new WindowsFileSystem(this, System.getProperty(USER_DIR));
    }

    @Override
    public final FileSystem getFileSystem(URI uri) {
        checkUri(uri);
        return theFileSystem;
    }
}
```

windows平台上关于 FileSystem 的实现： openjdk\jdk\src\windows\classes\sun\nio\fs

linux平台上关于 FileSystem 的实现：openjdk\jdk\src\solaris\classes\sun\nio\fs

* WindowsFileSystem

	openjdk\jdk\src\windows\classes\sun\nio\fs\WindowsFileSystem.java

	WindowsFileSystemProvider： openjdk\jdk\src\windows\classes\sun\nio\fs\WindowsFileSystemProvider.java

* LinuxFileSystem

	openjdk\jdk\src\solaris\classes\sun\nio\fs\LinuxFileSystem.java

	LinuxFileSystemProvider： openjdk\jdk\src\solaris\classes\sun\nio\fs\LinuxFileSystemProvider.java

### java.nio.files 包的使用

#### 1. 获得 FileSystem

``` java
FileSystem fileSystem = FileSystems.getDefault();
```

#### 2. 打开已经存在的文件

``` java
// 1. 通过 File 类
File file = new File("file.txt");
Path p1 = file.toPath();

// 2. 通过 FileSystem 类
Path p2 = fileSystem.getPath("/file.txt");

// 3. 通过 Paths 类
Path p3 = Paths.getPath("/file.txt");
```

#### 3. 创建文件和目录

``` java
// 创建文件
Path p4 = Paths.get("/temp/file.txt");
Files.createFile(p4);

// 创建目录
Path p5 = Paths.get("/temp/dir");
Files.createDirectories(p5);
```

#### 文件IO

* Reading All Bytes or Lines from a File

	* Files.readAllBytes
	* Files.readAllLines
	* Files.write
	
* Buffered I/O Methods for Text Files

	* Files.newBufferedReader
	* Files.newBufferedWriter
	
* Methods for Unbuffered Streams and Interoperable with java.io APIs

	* Files.newInputStream
	* Files.newOutputStream
	
* Methods for Channels and ByteBuffers

	* Files.newByteChannel
	
* FileChannel

	* FileChannel.open

[Basic I/O](https://docs.oracle.com/javase/tutorial/essential/io/index.html)

使用 NIO.2 进行文件 IO： [Reading, Writing, and Creating Files](https://docs.oracle.com/javase/tutorial/essential/io/file.html)
