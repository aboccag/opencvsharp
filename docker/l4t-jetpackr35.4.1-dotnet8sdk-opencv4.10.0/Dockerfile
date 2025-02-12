# -------------------------------------------------------------------
# Stage 1: Builder – install dependencies, .NET SDK, build OpenCV and OpenCvSharp
# -------------------------------------------------------------------
FROM nvcr.io/nvidia/l4t-jetpack:r35.4.1 AS builder

# Set noninteractive mode and OpenCV version
ENV DEBIAN_FRONTEND=noninteractive
ENV OPENCV_VERSION=4.10.0

# Install build tools, libraries, and prerequisites
RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-transport-https \
    software-properties-common \
    wget \
    unzip \
    ca-certificates \
    build-essential \
    cmake \
    git \
    libtbb-dev \
    libatlas-base-dev \
    libgtk2.0-dev \
    libavcodec-dev \
    libavformat-dev \
    libswscale-dev \
    libdc1394-dev \
    libv4l-dev \
    libtheora-dev \
    libvorbis-dev \
    libxvidcore-dev \
    libopencore-amrnb-dev \
    libopencore-amrwb-dev \
    x264 \
    libtesseract-dev \
    libgdiplus \
    gnupg \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install .NET 8 SDK using the dotnet-install script (ARM-friendly)
RUN wget https://dot.net/v1/dotnet-install.sh -O dotnet-install.sh && \
    chmod +x dotnet-install.sh && \
    ./dotnet-install.sh --channel 8.0 --install-dir /root/.dotnet && \
    rm dotnet-install.sh

# Set environment variables for .NET
ENV DOTNET_ROOT=/root/.dotnet
ENV PATH="${DOTNET_ROOT}:${PATH}"

# ----- Build OpenCV from source -----
WORKDIR /opencv_build

# Download OpenCV and opencv_contrib source archives, unzip and rename directories
RUN wget -q https://github.com/opencv/opencv/archive/${OPENCV_VERSION}.zip && \
    unzip -q ${OPENCV_VERSION}.zip && rm ${OPENCV_VERSION}.zip && \
    mv opencv-${OPENCV_VERSION} opencv && \
    wget -q https://github.com/opencv/opencv_contrib/archive/${OPENCV_VERSION}.zip && \
    unzip -q ${OPENCV_VERSION}.zip && rm ${OPENCV_VERSION}.zip && \
    mv opencv_contrib-${OPENCV_VERSION} opencv_contrib

# Build OpenCV (using static libraries in this example)
WORKDIR /opencv_build/opencv
RUN mkdir build && cd build && \
    cmake -DOPENCV_EXTRA_MODULES_PATH=/opencv_build/opencv_contrib/modules \
          -DCMAKE_BUILD_TYPE=RELEASE \
          -DBUILD_SHARED_LIBS=OFF \
          -DENABLE_CXX11=ON \
          -DBUILD_EXAMPLES=OFF \
          -DWITH_GSTREAMER=OFF \
          -DOPENCV_ENABLE_NONFREE=ON \
          .. && \
    make -j"$(nproc)" && make install && ldconfig

# ----- Build OpenCvSharp native (C++ wrapper) -----
WORKDIR /opencvsharp
RUN git clone https://github.com/aboccag/opencvsharp.git . && \
    mkdir build && cd build && \
    cmake -D CMAKE_INSTALL_PREFIX=/opencvsharp/build ../src && \
    make -j"$(nproc)" && make install && \
    cp /opencvsharp/build/OpenCvSharpExtern/libOpenCvSharpExtern.so /usr/lib/

# (Optional) Test the native C++ build by compiling a small test program
RUN echo "\
#include <stdio.h>\n\
int core_Mat_sizeof();\n\
int main(){\n\
  int i = core_Mat_sizeof();\n\
  printf(\"sizeof(Mat) = %d\", i);\n\
  return 0;\n\
}" > /test.c && \
gcc -I. -L. /test.c -o test_bin -lOpenCvSharpExtern && \
LD_LIBRARY_PATH=. ./test_bin


# ----- Build the C# part of OpenCvSharp -----
RUN git clone https://github.com/aboccag/opencvsharp.git . && \
    cd src/OpenCvSharp && dotnet build -c Release -f net8.0 && \
    cd ../OpenCvSharp.Extensions && dotnet build -c Release -f net8.0 && \
    dotnet test ../../test/OpenCvSharp.Tests/OpenCvSharp.Tests.csproj -c Release -f net8.0 --logger "trx;LogFileName=test-results.trx" || true

# -------------------------------------------------------------------
# Stage 2: Final runtime image – install .NET runtime via dotnet-install.sh and copy built libraries
# -------------------------------------------------------------------
FROM nvcr.io/nvidia/l4t-jetpack:r35.4.1

ENV DEBIAN_FRONTEND=noninteractive

# Install minimal prerequisites plus additional dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    apt-transport-https \
    software-properties-common \
    wget \
    ca-certificates \
    gnupg \
    bash \
    libtesseract4 \
    libdc1394-25 \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install .NET 8 runtime (ASP.NET Core) using dotnet-install.sh
RUN wget https://dot.net/v1/dotnet-install.sh -O dotnet-install.sh && \
    chmod +x dotnet-install.sh && \
    ./dotnet-install.sh --channel 8.0 --runtime aspnetcore --install-dir /root/.dotnet && \
    rm dotnet-install.sh

# Set environment variables for .NET and the dynamic linker
ENV DOTNET_ROOT=/root/.dotnet
ENV PATH="${DOTNET_ROOT}:${PATH}"
ENV LD_LIBRARY_PATH="/usr/lib:${LD_LIBRARY_PATH}"

# Copy the native libraries built in the builder stage
COPY --from=builder /usr/lib/libOpenCvSharpExtern.so /usr/lib/
COPY --from=builder /usr/local/lib/libopencv_*.a /usr/local/lib/

WORKDIR /app

CMD [ "bash" ]
