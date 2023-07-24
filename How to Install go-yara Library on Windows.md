## How to Install go-yara Library on Windows

Go-Yara (github.com/hillu/go-yara/v4) is a powerful Go (Golang) library that allows you to work with YARA rules and scan files for patterns and signatures efficiently. In this guide, we will walk you through the steps required to install the go-yara library on a Windows machine using MSYS2 and MinGW.

## Prerequisites

Before we begin, make sure you have the following prerequisites installed on your Windows machine:

 1. MSYS2: You can download the MSYS2 installer from the official website and follow the installation instructions provided there.

 2. Go: Make sure you have Go (Golang) installed on your machine. You can download the latest version of Go from the official website.

## Installation Steps

### Step 1: Install required packages

Open the MSYS2 MinGW 64-bit terminal and run the following command to install the necessary packages:

    pacman -S mingw-w64-x86_64-toolchain mingw-w64-x86_64-gcc mingw-w64-x86_64-make mingw-w64-x86_64-pkg-config base-devel openssl-devel autoconf-wrapper automake libtool

### Step 2: Download YARA source code

Download the YARA source code from the official GitHub repository at: [https://github.com/VirusTotal/yara/releases](https://github.com/VirusTotal/yara/releases)

### Step 3: Build and Install YARA

Navigate to the YARA source code directory using the MSYS2 terminal:

    cd /path/to/yara-source-code

Next, run the following commands to build and install YARA:

    ./bootstrap.sh
    ./configure
    make
    make install

### Step 4: Add MinGW to Path

To ensure that everything works smoothly, you need to add the MinGW binaries directory to your system’s Path. By default, MinGW is installed at C:\msys64\mingw64\bin. Add this path to your system's environment variables.

### Step 5: Run your application

Now you can run your application from any command line or IDE directly (you don’t need to use MinGW anymore).

To make the go-yara library work correctly, you need to set the CGO_ENABLED and GOARCH environment variables. Open your system's Environment Variables settings and add the following entries:

    export CGO_ENABLED=1
    export GOARCH=amd64

When running your application you should use following command:

    go build -ldflags "-extldflags=-static" -tags yara_static main.go

## Test the Installation

To verify that go-yara is correctly installed on your system, you can create a simple Go program that uses the library to run a YARA rule. Here’s a basic example:

    package main
    
    import (
     "fmt"
    
     "github.com/hillu/go-yara/v4"
    )
    
    func main() {
     c, err := yara.NewCompiler()
     if c == nil || err != nil {
      fmt.Println("Error to create compiler:", err)
      return
     }
     rule := `rule test {
        meta: 
         author = "Aviad Levy"
        strings:
         $str = "abc"
        condition:
         $str
       }`
     if err = c.AddString(rule, ""); err != nil {
      fmt.Println("Error adding YARA rule:", err)
      return
     }
     r, err := c.GetRules()
     if err != nil {
      fmt.Println("Error getting YARA rule:", err)
      return
     }
     var m yara.MatchRules
     err = r.ScanMem([]byte(" abc "), 0, 0, &m)
     if err != nil {
      fmt.Println("Error matching YARA rule:", err)
      return
     }
     fmt.Printf("Matches: %+v", m)
    }

Save this code to a file (e.g., main.go). Then, run the program using the go run command:

    go run -ldflags "-extldflags=-static" -tags yara_static main.go

If everything is set up correctly, you should see the matched YARA rule printed on the screen.

## Conclusion

Congratulations! You’ve successfully installed the go-yara library on your Windows machine and verified its functionality with a simple YARA rule scanning example. With this library, you can now take advantage of YARA’s powerful pattern matching capabilities in your Go projects.

For more information and detailed documentation on go-yara, be sure to check out the official GitHub repository:
[**GitHub - hillu/go-yara: Go bindings for YARA**
*Go bindings for YARA. Contribute to hillu/go-yara development by creating an account on GitHub.*github.com](https://github.com/hillu/go-yara)

I hope that you found this guide helpful. Thanks for reading!
