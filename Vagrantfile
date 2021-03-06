require 'date'

Vagrant.configure(2) do |config|
    config.vm.provider :virtualbox do |v|
        v.name = 'exa'
        v.memory = 1024
        v.cpus = 1
    end

    developer = 'ubuntu'


    # We use Ubuntu instead of Debian because the image comes with two-way
    # shared folder support by default.
    config.vm.box = 'ubuntu/xenial64'
    config.vm.hostname = 'exa'


    # Install the dependencies needed for exa to build, as quietly as
    # apt can do.
    config.vm.provision :shell, privileged: true, inline: <<-EOF
        set -xe
        apt-get install -qq -o=Dpkg::Use-Pty=0 -y \
          git cmake curl attr libgit2-dev \
          fish zsh bash bash-completion
    EOF


    # Guarantee that the timezone is UTC -- some of the tests
    # depend on this (for now).
    config.vm.provision :shell, privileged: true, inline:
        %[timedatectl set-timezone UTC]


    # Install Rust.
    # This is done as vagrant, not root, because it’s vagrant
    # who actually uses it. Sent to /dev/null because the progress
    # bar produces a ton of output.
    config.vm.provision :shell, privileged: false, inline:
        %[hash rustc &>/dev/null || curl -sSf https://static.rust-lang.org/rustup.sh | sh &> /dev/null]


    # Use a different ‘target’ directory on the VM than on the host.
    # By default it just uses the one in /vagrant/target, which can
    # cause problems if it has different permissions than the other
    # directories, or contains object files compiled for the host.
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        function put_line() {
          grep -q -F "$2" $1 || echo "$2" >> $1
        }

        put_line ~/.bashrc 'export CARGO_TARGET_DIR=/home/#{developer}/target'
        put_line ~/.bashrc 'export PATH=$PATH:/home/#{developer}/.cargo/bin'
    EOF


    # Create "dexa" and "rexa" scripts that run the debug and release
    # compiled versions of exa.
    config.vm.provision :shell, privileged: true, inline: <<-EOF
        set -xe

        echo -e "#!/bin/sh\n/home/#{developer}/target/debug/exa \\$*" > /usr/bin/exa
        echo -e "#!/bin/sh\n/home/#{developer}/target/release/exa \\$*" > /usr/bin/rexa
        chmod +x /usr/bin/{exa,rexa}
    EOF


    # Link the completion files so they’re “installed”.
    config.vm.provision :shell, privileged: true, inline: <<-EOF
        set -xe

        test -h /etc/bash_completion.d/exa \
          || ln -s /vagrant/contrib/completions.bash /etc/bash_completion.d/exa

        test -h /usr/share/zsh/vendor-completions/_exa \
          || ln -s /vagrant/contrib/completions.zsh /usr/share/zsh/vendor-completions/_exa

        test -h /usr/share/fish/completions/exa.fish \
          || ln -s /vagrant/contrib/completions.fish /usr/share/fish/completions/exa.fish
    EOF


    # We create two users that own the test files.
    # The first one just owns the ordinary ones, because we don’t want the
    # test outputs to depend on “vagrant” or “ubuntu” existing.
    user = "cassowary"
    config.vm.provision :shell, privileged: true, inline:
        %[id -u #{user} &>/dev/null || useradd #{user}]


    # The second one has a long name, to test that the file owner column
    # widens correctly. The benefit of Vagrant is that we don’t need to
    # set this up on the *actual* system!
    longuser = "antidisestablishmentarienism"
    config.vm.provision :shell, privileged: true, inline:
        %[id -u #{longuser} &>/dev/null || useradd #{longuser}]


    # Because the timestamps are formatted differently depending on whether
    # they’re in the current year or not (see `details.rs`), we have to make
    # sure that the files are created in the current year, so they get shown
    # in the format we expect.
    current_year = Date.today.year
    some_date = "#{current_year}01011234.56"  # 1st January, 12:34:56


    # We also need an UID and a GID that are guaranteed to not exist, to
    # test what happen when they don’t.
    invalid_uid = 666
    invalid_gid = 616


    # Delete old testcases if they exist already, then create a
    # directory to house new ones.
    test_dir = "/testcases"
    config.vm.provision :shell, privileged: true, inline: <<-EOF
        set -xe
        rm -rfv #{test_dir}
        mkdir #{test_dir}
        chmod 777 #{test_dir}
    EOF


    # Awkward file size testcases.
    # This needs sudo to set the files’ users at the very end.
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        set -xe
        mkdir "#{test_dir}/files"
        for i in {1..13}; do
            fallocate -l "$i" "#{test_dir}/files/$i"_bytes
            fallocate -l "$i"KiB "#{test_dir}/files/$i"_KiB
            fallocate -l "$i"MiB "#{test_dir}/files/$i"_MiB
        done

        touch -t #{some_date} "#{test_dir}/files/"*
        chmod 644 "#{test_dir}/files/"*
        sudo chown #{user}:#{user} "#{test_dir}/files/"*
    EOF


    # File name extension testcases.
    # These are tested in grid view, so we don’t need to bother setting
    # owners or timestamps or anything.
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        set -xe
        mkdir "#{test_dir}/file-names-exts"

        touch "#{test_dir}/file-names-exts/Makefile"

        touch "#{test_dir}/file-names-exts/image.png"
        touch "#{test_dir}/file-names-exts/image.svg"

        touch "#{test_dir}/file-names-exts/video.avi"
        touch "#{test_dir}/file-names-exts/video.wmv"

        touch "#{test_dir}/file-names-exts/music.mp3"
        touch "#{test_dir}/file-names-exts/music.ogg"

        touch "#{test_dir}/file-names-exts/lossless.flac"
        touch "#{test_dir}/file-names-exts/lossless.wav"

        touch "#{test_dir}/file-names-exts/crypto.asc"
        touch "#{test_dir}/file-names-exts/crypto.signature"

        touch "#{test_dir}/file-names-exts/document.pdf"
        touch "#{test_dir}/file-names-exts/document.xlsx"

        touch "#{test_dir}/file-names-exts/compressed.zip"
        touch "#{test_dir}/file-names-exts/compressed.tar.gz"
        touch "#{test_dir}/file-names-exts/compressed.tgz"

        touch "#{test_dir}/file-names-exts/backup~"
        touch "#{test_dir}/file-names-exts/#SAVEFILE#"
        touch "#{test_dir}/file-names-exts/file.tmp"

        touch "#{test_dir}/file-names-exts/compiled.class"
        touch "#{test_dir}/file-names-exts/compiled.o"
        touch "#{test_dir}/file-names-exts/compiled.js"
        touch "#{test_dir}/file-names-exts/compiled.coffee"
    EOF


    # File name testcases.
    # bash really doesn’t want you to create a file with escaped characters
    # in its name, so we have to resort to the echo builtin and touch!
    #
    # The double backslashes are not strictly necessary; without them, Ruby
    # will interpolate them instead of bash, but because Vagrant prints out
    # each command it runs, your *own* terminal will go “ding” from the alarm!
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        set -xe
        mkdir "#{test_dir}/file-names"

        echo -ne "#{test_dir}/file-names/ascii: hello" | xargs -0 touch
        echo -ne "#{test_dir}/file-names/emoji: [🆒]"  | xargs -0 touch
        echo -ne "#{test_dir}/file-names/utf-8: pâté"  | xargs -0 touch

        echo -ne "#{test_dir}/file-names/bell: [\\a]"         | xargs -0 touch
        echo -ne "#{test_dir}/file-names/backspace: [\\b]"    | xargs -0 touch
        echo -ne "#{test_dir}/file-names/form-feed: [\\f]"    | xargs -0 touch
        echo -ne "#{test_dir}/file-names/new-line: [\\n]"     | xargs -0 touch
        echo -ne "#{test_dir}/file-names/return: [\\r]"       | xargs -0 touch
        echo -ne "#{test_dir}/file-names/tab: [\\t]"          | xargs -0 touch
        echo -ne "#{test_dir}/file-names/vertical-tab: [\\v]" | xargs -0 touch

        echo -ne "#{test_dir}/file-names/escape: [\\033]"               | xargs -0 touch
        echo -ne "#{test_dir}/file-names/ansi: [\\033[34mblue\\033[0m]" | xargs -0 touch

        echo -ne "#{test_dir}/file-names/invalid-utf8-1: [\\xFF]"                | xargs -0 touch
        echo -ne "#{test_dir}/file-names/invalid-utf8-2: [\\xc3\\x28]"           | xargs -0 touch
        echo -ne "#{test_dir}/file-names/invalid-utf8-3: [\\xe2\\x82\\x28]"      | xargs -0 touch
        echo -ne "#{test_dir}/file-names/invalid-utf8-4: [\\xf0\\x28\\x8c\\x28]" | xargs -0 touch

        echo -ne "#{test_dir}/file-names/new-line-dir: [\\n]"                | xargs -0 mkdir
        echo -ne "#{test_dir}/file-names/new-line-dir: [\\n]/subfile"        | xargs -0 touch
        echo -ne "#{test_dir}/file-names/new-line-dir: [\\n]/another: [\\n]" | xargs -0 touch
        echo -ne "#{test_dir}/file-names/new-line-dir: [\\n]/broken"         | xargs -0 touch

        mkdir "#{test_dir}/file-names/links"
        ln -s "#{test_dir}/file-names/new-line-dir"*/* "#{test_dir}/file-names/links"

        echo -ne "#{test_dir}/file-names/new-line-dir: [\\n]/broken" | xargs -0 rm
    EOF


    # Special file testcases.
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        set -xe
        mkdir "#{test_dir}/specials"

        sudo mknod "#{test_dir}/specials/block-device" b  3 60
        sudo mknod "#{test_dir}/specials/char-device"  c 14 40
        sudo mknod "#{test_dir}/specials/named-pipe"   p

        sudo touch -t #{some_date} "#{test_dir}/specials/"*
    EOF


    # Awkward symlink testcases.
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        set -xe
        mkdir "#{test_dir}/links"

        ln -s /            "#{test_dir}/links/root"
        ln -s /usr         "#{test_dir}/links/usr"
        ln -s nowhere      "#{test_dir}/links/broken"
        ln -s /proc/1/root "#{test_dir}/links/forbidden"

        touch "#{test_dir}/links/some_file"
        ln -s "#{test_dir}/links/some_file" "#{test_dir}/links/some_file_absolute"
        (cd "#{test_dir}/links"; ln -s "some_file" "some_file_relative")
        (cd "#{test_dir}/links"; ln -s "."         "current_dir")
        (cd "#{test_dir}/links"; ln -s ".."        "parent_dir")
        (cd "#{test_dir}/links"; ln -s "itself"    "itself")
    EOF


    # Awkward passwd testcases.
    # sudo is needed for these because we technically aren’t a member
    # of the groups (because they don’t exist), and chown and chgrp
    # are smart enough to disallow it!
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        set -xe
        mkdir "#{test_dir}/passwd"

        touch -t #{some_date}             "#{test_dir}/passwd/unknown-uid"
        chmod 644                         "#{test_dir}/passwd/unknown-uid"
        sudo chown #{invalid_uid}:#{user} "#{test_dir}/passwd/unknown-uid"

        touch -t #{some_date}             "#{test_dir}/passwd/unknown-gid"
        chmod 644                         "#{test_dir}/passwd/unknown-gid"
        sudo chown #{user}:#{invalid_gid} "#{test_dir}/passwd/unknown-gid"
    EOF


    # Awkward permission testcases.
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        set -xe
        mkdir "#{test_dir}/permissions"

        touch     "#{test_dir}/permissions/all-permissions"
        chmod 777 "#{test_dir}/permissions/all-permissions"

        touch     "#{test_dir}/permissions/no-permissions"
        chmod 000 "#{test_dir}/permissions/no-permissions"

        mkdir     "#{test_dir}/permissions/forbidden-directory"
        chmod 000 "#{test_dir}/permissions/forbidden-directory"

        for perms in 001 002 004 010 020 040 100 200 400; do
            touch        "#{test_dir}/permissions/$perms"
            chmod $perms "#{test_dir}/permissions/$perms"
        done

        touch -t #{some_date}      "#{test_dir}/permissions/"*
        sudo chown #{user}:#{user} "#{test_dir}/permissions/"*
    EOF


    # Awkward extended attribute testcases.
    config.vm.provision :shell, privileged: false, inline: <<-EOF
        set -xe
        mkdir "#{test_dir}/attributes"

        touch "#{test_dir}/attributes/none"

        touch "#{test_dir}/attributes/one"
        setfattr -n user.greeting -v hello "#{test_dir}/attributes/one"

        touch "#{test_dir}/attributes/two"
        setfattr -n user.greeting -v hello "#{test_dir}/attributes/two"
        setfattr -n user.another_greeting -v hi "#{test_dir}/attributes/two"

        #touch "#{test_dir}/attributes/forbidden"
        #setfattr -n user.greeting -v hello "#{test_dir}/attributes/forbidden"
        #chmod +a "$YOU deny readextattr" "#{test_dir}/attributes/forbidden"

        mkdir "#{test_dir}/attributes/dirs"

        mkdir "#{test_dir}/attributes/dirs/empty-with-attribute"
        setfattr -n user.greeting -v hello "#{test_dir}/attributes/dirs/empty-with-attribute"

        mkdir "#{test_dir}/attributes/dirs/full-with-attribute"
        touch "#{test_dir}/attributes/dirs/full-with-attribute/file"
        setfattr -n user.greeting -v hello "#{test_dir}/attributes/dirs/full-with-attribute"

        mkdir "#{test_dir}/attributes/dirs/full-but-forbidden"
        touch "#{test_dir}/attributes/dirs/full-but-forbidden/file"
        #setfattr -n user.greeting -v hello "#{test_dir}/attributes/dirs/full-but-forbidden"
        #chmod 000 "#{test_dir}/attributes/dirs/full-but-forbidden"
        #chmod +a "$YOU deny readextattr" "#{test_dir}/attributes/dirs/full-but-forbidden"

        touch -t #{some_date} "#{test_dir}/attributes"
        touch -t #{some_date} "#{test_dir}/attributes/"*
        touch -t #{some_date} "#{test_dir}/attributes/dirs/"*
        touch -t #{some_date} "#{test_dir}/attributes/dirs/"*/*

        sudo chown #{user}:#{user} -R "#{test_dir}/attributes"
    EOF


    # Install kcov for test coverage
    # This doesn’t run coverage over the xtests so it’s less useful for now
    if ENV.key?('INSTALL_KCOV')
        config.vm.provision :shell, privileged: false, inline: <<-EOF
            set -xe

            test -e ~/.cargo/bin/cargo-kcov \
              || cargo install cargo-kcov

            sudo apt-get install -qq -o=Dpkg::Use-Pty=0 -y \
              cmake g++ pkg-config \
              libcurl4-openssl-dev libdw-dev binutils-dev libiberty-dev

            cargo kcov --print-install-kcov-sh | sudo sh
        EOF
    end
end
