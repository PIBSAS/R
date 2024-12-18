# Recalbox con GitHub Actions (Requiere Enterprise)

name: Build and Release Recalbox

permissions:
  contents: write
on:
  push:
    branches:
      - main  # Trigger on pushes to the main branch
  workflow_dispatch:  # Allow manual triggering

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout source code
      uses: actions/checkout@v4

    - name: Set up Docker
      run: |
        sudo apt-get update
        sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
        sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
        sudo apt-get update
        sudo apt-get install -y docker-ce docker-ce-cli containerd.io
        sudo usermod -aG docker $USER

    - name: Build Recalbox
      env:
        ARCH: rpi4_64  # Change this to your target architecture
        DOCKER_FLAGS: ""
        RECALBOX_VERSION: 9.0  # Example version, update as needed
      run: |
        git clone https://gitlab.com/recalbox/recalbox.git recalbox-${ARCH}
        cd recalbox-${ARCH}
        git submodule update --init --recursive
        #sed -i 's/-ti//g' ./scripts/linux/recaldocker.sh
        ./scripts/linux/recaldocker.sh make recalbox-${ARCH}_defconfig
        if [ -f "/home/runner/work/R/R/recalbox-rpi4_64/output/.config" ]; then
          echo "Buildroot config found."
        else
          echo "Buildroot config not found, exiting."
          exit 1
        fi
        make -C /home/runner/work/R/R/recalbox-rpi4_64/buildroot O=/home/runner/work/R/R/recalbox-rpi4_64/output

   # - name: Debug Buildroot Directory
    #  run: |
     #   echo "Listing Buildroot Directory"
      #  ls -la recalbox-${ARCH}/buildroot
       # echo "Listing output directory"
        #ls -la recalbox-${ARCH}/output
    
    #- name: Recreate Buildroot Directory
     # run: |
      #  cd recalbox-${ARCH}
       # ./scripts/linux/recaldocker.sh make recalbox-${ARCH}_defconfig
        #ls -la buildroot  # Verify if buildroot is created
        
    #- name: Debug MergeToBR
     # run: |
      #  cd recalbox-${ARCH}
       # sed -i '/Check environment/a set -x' ./scripts/linux/mergeToBR.sh
        #./scripts/linux/mergeToBR.sh
        
    - name: Upload Recalbox image artifact
      uses: actions/upload-artifact@v3
      with:
        name: recalbox-image
        path: recalbox-${{ env.ARCH }}/output/images/recalbox.img

  release:
    needs: build
    runs-on: ubuntu-latest

    steps:
    - name: Download build artifact
      uses: actions/download-artifact@v3
      with:
        name: recalbox-image

    - name: Create Release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: "recalbox-${{ env.ARCH }}-${{ env.RECALBOX_VERSION }}"
        release_name: "Recalbox Release ${{ env.ARCH }} - ${{ env.RECALBOX_VERSION }}"
        body: |
          This release contains the compiled Recalbox image for architecture ${{ env.ARCH }}.
        draft: false
        prerelease: false

    - name: Upload Release Asset
      uses: actions/upload-release-asset@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        upload_url: ${{ steps.create-release.outputs.upload_url }}
        asset_path: recalbox-${{ env.ARCH }}/output/images/recalbox.img
        asset_name: recalbox-${{ env.ARCH }}-${{ env.RECALBOX_VERSION }}.img
        asset_content_type: application/octet-stream
