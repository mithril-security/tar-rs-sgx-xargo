# tar-rs

[Documentation](https://docs.rs/tar)

A tar archive reading/writing library for Rust.

```toml
# Cargo.toml
[dependencies]
tar = "0.4"
```

## SGX related

When compiled in an SGX project, this port disable all access to the filesystem, as Intel SGX consider the filesystem as unsafe.

It is only possible to load a tar file from memory.

## Reading an archive (without SGX)

```rust,no_run
extern crate tar;

use std::io::prelude::*;
use std::fs::File;
use tar::Archive;

fn main() {
    let file = File::open("foo.tar").unwrap();
    let mut a = Archive::new(file);

    for file in a.entries().unwrap() {
        // Make sure there wasn't an I/O error
        let mut file = file.unwrap();

        // Inspect metadata about the file
        println!("{:?}", file.header().path().unwrap());
        println!("{}", file.header().size().unwrap());

        // files implement the Read trait
        let mut s = String::new();
        file.read_to_string(&mut s).unwrap();
        println!("{}", s);
    }
}

```

## Writing an archive (without SGX)

```rust,no_run
extern crate tar;

use std::io::prelude::*;
use std::fs::File;
use tar::Builder;

fn main() {
    let file = File::create("foo.tar").unwrap();
    let mut a = Builder::new(file);

    a.append_path("file1.txt").unwrap();
    a.append_file("file2.txt", &mut File::open("file3.txt").unwrap()).unwrap();
}
```

## Reading an archive (SGX - host part)

```rust,no_run
extern crate tar;

use std::io::prelude::*;
use std::fs::File;

fn main() -> std::io::Result<()> {
    let file = File::open("foo.tar").unwrap();
    let mut buffer = vec![0; metadata.len() as usize];
    file.read(&mut buffer).expect("buffer overflow");
    let mut array = &buffer[..];
    let mut retval = sgx_status_t::SGX_SUCCESS;

    let result = unsafe {
        read_tar(enclave.geteid(),
                      &mut retval,
                      array.as_ptr() as * const u8,
                      array.len())
    };

    match result {
        sgx_status_t::SGX_SUCCESS => {},
        _ => {
            println!("[-] ECALL Enclave Failed {}!", result.as_str());
            let error = Error::new(ErrorKind::Other, "ECALL failed");
            return Err(error);
        }
    }
    Ok(())
}

```

## Reading an archive (SGX - enclave part)

```rust,no_run
#![crate_name = "tar_test"]
#![crate_type = "staticlib"]

extern crate tar;

#![cfg_attr(not(target_env = "sgx"), no_std)]
#![cfg_attr(target_env = "sgx", feature(rustc_private))]

extern crate sgx_types;
#[cfg(not(target_env = "sgx"))]
#[macro_use]
extern crate sgx_tstd as std;

use std::io::prelude::*;
use std::io::Read;
use tar::Archive;

pub extern "C" fn read_tar(tar_ptr: *const u8, tar_len: usize) -> sgx_status_t 
{
    let mut tar_slice = unsafe { slice::from_raw_parts(tar_ptr, tar_len) };
    let mut a = Archive::new(tar_slice);

    for entry in a.entries() 
    {
            let mut entry = entry?;
            let path = entry.path()?.to_path_buf();
    }
    sgx_status_t::SGX_SUCCESS
}

```

# License

This project is licensed under either of

 * Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or
   http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or
   http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this project by you, as defined in the Apache-2.0 license,
shall be dual licensed as above, without any additional terms or conditions.
