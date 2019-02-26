[UPDATE] I'm updating this answer to work with **Internet Explorer 11**, in **Windows 10 x64** with **Visual Studio 2017 Community**.
The previous version of this answer (for Internet Explorer 8, in Windows 7 x64 and Visual Studio 2010) is at the bottom of this answer.

Creating a Working Internet Explorer 11 Add-on
----------------------------------------------

I am using **Visual Studio 2017 Community**, **C#**, **.Net Framework 4.6.1**, so some of these steps might be slightly different for you.

You need to **open Visual Studio as Administrator** to build the solution, so that the post-build script can register the BHO (needs registry access).

Start by creating a class library.
I called mine *InternetExplorerExtension*.

Add these COM references to the project:

 - Interop.SHDocVw
 - Microsoft.mshtml

*Note:* Somehow MSHTML was not registered in my system, even though I could find in in the Add Reference window. This caused an error while building:

> Cannot find wrapper assembly for type library "MSHTML"

The fix can be found at http://techninotes.blogspot.com/2016/08/fixing-cannot-find-wrapper-assembly-for.html
Or, you can run this batch script:

    "%ProgramFiles(x86)%\Microsoft Visual Studio\2017\Community\Common7\Tools\VsDevCmd.bat"
    cd "%ProgramFiles(x86)%\Microsoft Visual Studio\2017\Community\Common7\IDE\PublicAssemblies"
    regasm Microsoft.mshtml.dll
    gacutil /i Microsoft.mshtml.dll

Create the following files:

**IEAddon.cs**

    using System;
    using System.Collections.Generic;
    using System.Runtime.InteropServices;
    using System.Windows.Forms;
    using Microsoft.Win32;
    using mshtml;
    using SHDocVw;

    namespace InternetExplorerExtension
    {
        [ComVisible(true)]
        [ClassInterface(ClassInterfaceType.None)]
        [Guid("D40C654D-7C51-4EB3-95B2-1E23905C2A2D")]
        [ProgId("MyBHO.WordHighlighter")]
        public class WordHighlighterBHO : IObjectWithSite, IOleCommandTarget
        {
            const string DefaultTextToHighlight = "browser";

            IWebBrowser2 browser;
            private object site;

            #region Highlight Text
            void OnDocumentComplete(object pDisp, ref object URL)
            {
                try
                {
                    // @Eric Stob: Thanks for this hint!
                    // This was used to prevent this method being executed more than once in IE8... but now it seems to not work anymore.
                    //if (pDisp != this.site)
                    //    return;

                    var document2 = browser.Document as IHTMLDocument2;
                    var document3 = browser.Document as IHTMLDocument3;

                    var window = document2.parentWindow;
                    window.execScript(@"function FncAddedByAddon() { alert('Message added by addon.'); }");

                    Queue<IHTMLDOMNode> queue = new Queue<IHTMLDOMNode>();
                    foreach (IHTMLDOMNode eachChild in document3.childNodes)
                        queue.Enqueue(eachChild);

                    while (queue.Count > 0)
                    {
                        // replacing desired text with a highlighted version of it
                        var domNode = queue.Dequeue();

                        var textNode = domNode as IHTMLDOMTextNode;
                        if (textNode != null)
                        {
                            if (textNode.data.Contains(TextToHighlight))
                            {
                                var newText = textNode.data.Replace(TextToHighlight, "<span style='background-color: yellow; cursor: hand;' onclick='javascript:FncAddedByAddon()' title='Click to open script based alert window.'>" + TextToHighlight + "</span>");
                                var newNode = document2.createElement("span");
                                newNode.innerHTML = newText;
                                domNode.replaceNode((IHTMLDOMNode)newNode);
                            }
                        }
                        else
                        {
                            // adding children to collection
                            var x = (IHTMLDOMChildrenCollection)(domNode.childNodes);
                            foreach (IHTMLDOMNode eachChild in x)
                            {
                                if (eachChild is mshtml.IHTMLScriptElement)
                                    continue;
                                if (eachChild is mshtml.IHTMLStyleElement)
                                    continue;

                                queue.Enqueue(eachChild);
                            }
                        }
                    }
                }
                catch (Exception ex)
                {
                    MessageBox.Show(ex.Message);
                }
            }
            #endregion
            #region Load and Save Data
            static string TextToHighlight = DefaultTextToHighlight;
            public static string RegData = "Software\\MyIEExtension";

            [DllImport("ieframe.dll")]
            public static extern int IEGetWriteableHKCU(ref IntPtr phKey);

            private static void SaveOptions()
            {
                // In IE 7,8,9,(desktop)10 tabs run in Protected Mode
                // which prohibits writes to HKLM, HKCU.
                // Must ask IE for "Writable" registry section pointer
                // which will be something like HKU/S-1-7***/Software/AppDataLow/
                // In "metro" IE 10 mode, tabs run in "Enhanced Protected Mode"
                // where BHOs are not allowed to run, except in edge cases.
                // see http://blogs.msdn.com/b/ieinternals/archive/2012/03/23/understanding-ie10-enhanced-protected-mode-network-security-addons-cookies-metro-desktop.aspx
                IntPtr phKey = new IntPtr();
                var answer = IEGetWriteableHKCU(ref phKey);
                RegistryKey writeable_registry = RegistryKey.FromHandle(
                    new Microsoft.Win32.SafeHandles.SafeRegistryHandle(phKey, true)
                );
                RegistryKey registryKey = writeable_registry.OpenSubKey(RegData, true);

                if (registryKey == null)
                    registryKey = writeable_registry.CreateSubKey(RegData);
                registryKey.SetValue("Data", TextToHighlight);

                writeable_registry.Close();
            }
            private static void LoadOptions()
            {
                // In IE 7,8,9,(desktop)10 tabs run in Protected Mode
                // which prohibits writes to HKLM, HKCU.
                // Must ask IE for "Writable" registry section pointer
                // which will be something like HKU/S-1-7***/Software/AppDataLow/
                // In "metro" IE 10 mode, tabs run in "Enhanced Protected Mode"
                // where BHOs are not allowed to run, except in edge cases.
                // see http://blogs.msdn.com/b/ieinternals/archive/2012/03/23/understanding-ie10-enhanced-protected-mode-network-security-addons-cookies-metro-desktop.aspx
                IntPtr phKey = new IntPtr();
                var answer = IEGetWriteableHKCU(ref phKey);
                RegistryKey writeable_registry = RegistryKey.FromHandle(
                    new Microsoft.Win32.SafeHandles.SafeRegistryHandle(phKey, true)
                );
                RegistryKey registryKey = writeable_registry.OpenSubKey(RegData, true);

                if (registryKey == null)
                    registryKey = writeable_registry.CreateSubKey(RegData);
                registryKey.SetValue("Data", TextToHighlight);

                if (registryKey == null)
                {
                    TextToHighlight = DefaultTextToHighlight;
                }
                else
                {
                    TextToHighlight = (string)registryKey.GetValue("Data");
                }
                writeable_registry.Close();
            }
            #endregion

            [Guid("6D5140C1-7436-11CE-8034-00AA006009FA")]
            [InterfaceType(1)]
            public interface IServiceProvider
            {
                int QueryService(ref Guid guidService, ref Guid riid, out IntPtr ppvObject);
            }

            #region Implementation of IObjectWithSite
            int IObjectWithSite.SetSite(object site)
            {
                this.site = site;

                if (site != null)
                {
                    LoadOptions();

                    var serviceProv = (IServiceProvider)this.site;
                    var guidIWebBrowserApp = Marshal.GenerateGuidForType(typeof(IWebBrowserApp)); // new Guid("0002DF05-0000-0000-C000-000000000046");
                    var guidIWebBrowser2 = Marshal.GenerateGuidForType(typeof(IWebBrowser2)); // new Guid("D30C1661-CDAF-11D0-8A3E-00C04FC9E26E");
                    IntPtr intPtr;
                    serviceProv.QueryService(ref guidIWebBrowserApp, ref guidIWebBrowser2, out intPtr);

                    browser = (IWebBrowser2)Marshal.GetObjectForIUnknown(intPtr);

                    ((DWebBrowserEvents2_Event)browser).DocumentComplete +=
                        new DWebBrowserEvents2_DocumentCompleteEventHandler(this.OnDocumentComplete);
                }
                else
                {
                    ((DWebBrowserEvents2_Event)browser).DocumentComplete -=
                        new DWebBrowserEvents2_DocumentCompleteEventHandler(this.OnDocumentComplete);
                    browser = null;
                }
                return 0;
            }
            int IObjectWithSite.GetSite(ref Guid guid, out IntPtr ppvSite)
            {
                IntPtr punk = Marshal.GetIUnknownForObject(browser);
                int hr = Marshal.QueryInterface(punk, ref guid, out ppvSite);
                Marshal.Release(punk);
                return hr;
            }
            #endregion
            #region Implementation of IOleCommandTarget
            int IOleCommandTarget.QueryStatus(IntPtr pguidCmdGroup, uint cCmds, ref OLECMD prgCmds, IntPtr pCmdText)
            {
                return 0;
            }
            int IOleCommandTarget.Exec(IntPtr pguidCmdGroup, uint nCmdID, uint nCmdexecopt, IntPtr pvaIn, IntPtr pvaOut)
            {
                try
                {
                    // Accessing the document from the command-bar.
                    var document = browser.Document as IHTMLDocument2;
                    var window = document.parentWindow;
                    var result = window.execScript(@"alert('You will now be allowed to configure the text to highlight...');");

                    var form = new HighlighterOptionsForm();
                    form.InputText = TextToHighlight;
                    if (form.ShowDialog() != DialogResult.Cancel)
                    {
                        TextToHighlight = form.InputText;
                        SaveOptions();
                    }
                }
                catch (Exception ex)
                {
                    MessageBox.Show(ex.Message);
                }

                return 0;
            }
            #endregion

            #region Registering with regasm
            public static string RegBHO = "Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\Browser Helper Objects";
            public static string RegCmd = "Software\\Microsoft\\Internet Explorer\\Extensions";

            [ComRegisterFunction]
            public static void RegisterBHO(Type type)
            {
                string guid = type.GUID.ToString("B");

                // BHO
                {
                    RegistryKey registryKey = Registry.LocalMachine.OpenSubKey(RegBHO, true);
                    if (registryKey == null)
                        registryKey = Registry.LocalMachine.CreateSubKey(RegBHO);
                    RegistryKey key = registryKey.OpenSubKey(guid);
                    if (key == null)
                        key = registryKey.CreateSubKey(guid);
                    key.SetValue("Alright", 1);
                    registryKey.Close();
                    key.Close();
                }

                // Command
                {
                    RegistryKey registryKey = Registry.LocalMachine.OpenSubKey(RegCmd, true);
                    if (registryKey == null)
                        registryKey = Registry.LocalMachine.CreateSubKey(RegCmd);
                    RegistryKey key = registryKey.OpenSubKey(guid);
                    if (key == null)
                        key = registryKey.CreateSubKey(guid);
                    key.SetValue("ButtonText", "Highlighter options");
                    key.SetValue("CLSID", "{1FBA04EE-3024-11d2-8F1F-0000F87ABD16}");
                    key.SetValue("ClsidExtension", guid);
                    key.SetValue("Icon", "");
                    key.SetValue("HotIcon", "");
                    key.SetValue("Default Visible", "Yes");
                    key.SetValue("MenuText", "&Highlighter options");
                    key.SetValue("ToolTip", "Highlighter options");
                    //key.SetValue("KeyPath", "no");
                    registryKey.Close();
                    key.Close();
                }
            }

            [ComUnregisterFunction]
            public static void UnregisterBHO(Type type)
            {
                string guid = type.GUID.ToString("B");
                // BHO
                {
                    RegistryKey registryKey = Registry.LocalMachine.OpenSubKey(RegBHO, true);
                    if (registryKey != null)
                        registryKey.DeleteSubKey(guid, false);
                }
                // Command
                {
                    RegistryKey registryKey = Registry.LocalMachine.OpenSubKey(RegCmd, true);
                    if (registryKey != null)
                        registryKey.DeleteSubKey(guid, false);
                }
            }
            #endregion
        }
    }

**Interop.cs**

    using System;
    using System.Runtime.InteropServices;
    namespace InternetExplorerExtension
    {
        [ComVisible(true)]
        [InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
        [Guid("FC4801A3-2BA9-11CF-A229-00AA003D7352")]
        public interface IObjectWithSite
        {
            [PreserveSig]
            int SetSite([MarshalAs(UnmanagedType.IUnknown)]object site);
            [PreserveSig]
            int GetSite(ref Guid guid, [MarshalAs(UnmanagedType.IUnknown)]out IntPtr ppvSite);
        }
    
    
        [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
        public struct OLECMDTEXT
        {
            public uint cmdtextf;
            public uint cwActual;
            public uint cwBuf;
            [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 100)]
            public char rgwz;
        }
    
        [StructLayout(LayoutKind.Sequential)]
        public struct OLECMD
        {
            public uint cmdID;
            public uint cmdf;
        }
    
        [ComImport(), ComVisible(true),
        Guid("B722BCCB-4E68-101B-A2BC-00AA00404770"),
        InterfaceTypeAttribute(ComInterfaceType.InterfaceIsIUnknown)]
        public interface IOleCommandTarget
        {
    
            [return: MarshalAs(UnmanagedType.I4)]
            [PreserveSig]
            int QueryStatus(
                [In] IntPtr pguidCmdGroup,
                [In, MarshalAs(UnmanagedType.U4)] uint cCmds,
                [In, Out, MarshalAs(UnmanagedType.Struct)] ref OLECMD prgCmds,
                //This parameter must be IntPtr, as it can be null
                [In, Out] IntPtr pCmdText);
    
            [return: MarshalAs(UnmanagedType.I4)]
            [PreserveSig]
            int Exec(
                //[In] ref Guid pguidCmdGroup,
                //have to be IntPtr, since null values are unacceptable
                //and null is used as default group!
                [In] IntPtr pguidCmdGroup,
                [In, MarshalAs(UnmanagedType.U4)] uint nCmdID,
                [In, MarshalAs(UnmanagedType.U4)] uint nCmdexecopt,
                [In] IntPtr pvaIn,
                [In, Out] IntPtr pvaOut);
        }
    }

and finally a form, that we will use to configure the options. In this form place a `TextBox` and an Ok `Button`. Set the **DialogResult** of the button to **Ok**. Place this code in the form code:

    using System.Windows.Forms;
    namespace InternetExplorerExtension
    {
        public partial class HighlighterOptionsForm : Form
        {
            public HighlighterOptionsForm()
            {
                InitializeComponent();
            }
    
            public string InputText
            {
                get { return this.textBox1.Text; }
                set { this.textBox1.Text = value; }
            }
        }
    }

In the project properties, do the following:

 - Sign the assembly with a strong-key;
 - In the Debug tab, set **Start External Program** to `C:\Program Files (x86)\Internet Explorer\iexplore.exe`
 - In the Debug tab, set **Command Line Arguments** to `http://msdn.microsoft.com/en-us/library/ms976373.aspx#bho_getintouch`
 - In the Build Events tab, set **Post-build events command line** to:

    <pre>"%ProgramFiles(x86)%\Microsoft SDKs\Windows\v10.0A\bin\NETFX 4.6.1 Tools\gacutil.exe" /f /i "$(TargetDir)$(TargetFileName)"

    "%windir%\Microsoft.NET\Framework\v4.0.30319\RegAsm.exe" /unregister "$(TargetDir)$(TargetFileName)"

    "%windir%\Microsoft.NET\Framework\v4.0.30319\RegAsm.exe" "$(TargetDir)$(TargetFileName)"</pre>

**Attention:** even though my computer is x64, I used the path of the non-x64 `gacutil.exe` and it worked... the one specific for x64 is at:

<pre>C:\Program Files (x86)\Microsoft SDKs\Windows\v10.0A\bin\NETFX 4.6.1 Tools\<b>x64\</b>gacutil.exe</pre>

**64bit IE** Needs 64bit-compiled and 64bit-registered BHO. Though I could only debug using 32bit IE11, the 32bit registered extension also worked by running 64bit IE11.

This answer appears to have some additional info about this: https://stackoverflow.com/a/23004613/195417

If you need to, you can use the 64bit regasm:

<pre>%windir%\Microsoft.NET\Framework<b>64\</b>v4.0.30319\RegAsm.exe</pre>

**How this add-on works**

I didn't change the behavior of the add-on... take a look at IE8 section bellow for description.

----
## Previous Answer for IE8
----

Man... this has been a lot of work!
I was so curious about how to do this, that I did it myself.

First of all... credit is not all mine. This is a compilation of what I found, on these sites:

 - [CodeProject article][1], how to make a BHO;
 - [15seconds][2], but it was not 15 seconds, it took about 7 hours;
 - [Microsoft tutorial][3], helped me adding the command button.
 - [And this social.msdn topic][4], that helped me figure out that the assembly must be in the GAC.
 - [This MSDN blog post][5] contains a fully-working example
 - many other sites, in the discovery process...

And of course, I wanted my answer to have the features you asked:

 - DOM traversal to find something;
 - a button that shows a window (in my case to setup)
 - persist the configuration (I will use registry for that)
 - and finally execute javascript.

I will describe it step by step, how I managed to do it working with **Internet Explorer 8**, in **Windows 7 x64**... note that I could not test in other configurations. Hope you understand =)



Creating a Working Internet Explorer 8 Add-on
---------------------------------------------

I am using **Visual Studio 2010**, **C# 4**, **.Net Framework 4**, so some of these steps might be slightly different for you.

Created a class library. I called mine *InternetExplorerExtension*.

Add these references to the project:

 - Interop.SHDocVw
 - Microsoft.mshtml

*Note: These references may be in different places in each computer.*

this is what my references section in csproj contains:

    <Reference Include="Interop.SHDocVw, Version=1.1.0.0, Culture=neutral, PublicKeyToken=90ba9c70f846762e, processorArchitecture=MSIL">
      <SpecificVersion>False</SpecificVersion>
      <EmbedInteropTypes>True</EmbedInteropTypes>
      <HintPath>C:\Program Files (x86)\Microsoft Visual Studio 9.0\Common7\IDE\PrivateAssemblies\Interop.SHDocVw.dll</HintPath>
    </Reference>
    <Reference Include="Microsoft.CSharp" />
    <Reference Include="Microsoft.mshtml, Version=7.0.3300.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a">
      <EmbedInteropTypes>True</EmbedInteropTypes>
    </Reference>
    <Reference Include="System" />
    <Reference Include="System.Data" />
    <Reference Include="System.Drawing" />
    <Reference Include="System.Windows.Forms" />
    <Reference Include="System.Xml" />

Create the files the same way as the updated IE11 files.

**IEAddon.cs**

You can uncomment the following lines from IE11 version:

    ...
    // @Eric Stob: Thanks for this hint!
    // This was used to prevent this method being executed more than once in IE8... but now it seems to not work anymore.
    if (pDisp != this.site)
        return;
    ...

**Interop.cs**

Same as IE11 version.

and finally a form, that we will use to configure the options. In this form place a `TextBox` and an Ok `Button`. Set the **DialogResult** of the button to **Ok**. The code is the same for IE11 addon.

In the project properties, do the following:

 - Sign the assembly with a strong-key;
 - In the Debug tab, set **Start External Program** to `C:\Program Files (x86)\Internet Explorer\iexplore.exe`
 - In the Debug tab, set **Command Line Arguments** to `http://msdn.microsoft.com/en-us/library/ms976373.aspx#bho_getintouch`
 - In the Build Events tab, set **Post-build events command line** to:

    <pre>"C:\Program Files (x86)\Microsoft SDKs\Windows\v7.0A\Bin\NETFX 4.0 Tools\<strong>x64</strong>\gacutil.exe" /f /i "$(TargetDir)$(TargetFileName)"

    "C:\Windows\Microsoft.NET\Framework\v4.0.30319\RegAsm.exe" /unregister "$(TargetDir)$(TargetFileName)"

    "C:\Windows\Microsoft.NET\Framework\v4.0.30319\RegAsm.exe" "$(TargetDir)$(TargetFileName)"</pre>

**Attention:** as my computer is x64, there is a specific x64 inside the path of gacutil executable on my machine that may be different on yours.

**64bit IE** Needs 64bit-compiled and 64bit-registered BHO. Use 64bit RegAsm.exe (usually lives in C:\Windows\Microsoft.NET\Framework64\v4.0.30319\RegAsm.exe) 

**How this add-on works**

It traverses all DOM tree, replacing the text, configured using the button, by itself with a yellow background. If you click on the yellowed texts, it calls a javascript function that was inserted on the page dynamically. The default word is 'browser', so that it matches a lot of them!
**EDIT:** after changing the string to be highlighted, you must click the URL box and press Enter... F5 will not work, I think that it is because F5 is considered as 'navigation', and it would require to listen to navigate event (maybe). I'll try to fix that later.


Now, it is time to go. I am very tired.
Feel free to ask questions... may be I will not be able to answer since I am going on a trip... in 3 days I'm back, but I'll try to come here in the meantime.

  [1]: http://www.codeproject.com/Articles/19971/How-to-attach-to-Browser-Helper-Object-BHO-with-C
  [2]: http://www.15seconds.com/issue/040331.htm
  [3]: http://msdn.microsoft.com/en-us/library/bb735854%28v=vs.85%29.aspx
  [4]: http://social.msdn.microsoft.com/Forums/en-US/ieextensiondevelopment/thread/74be2bc2-813e-4923-a6a0-42fd5757e30e/
  [5]: http://blogs.msdn.com/b/codefx/archive/2012/07/17/sample-of-july-17-create-ie-explorer-bar-in-c.aspx
