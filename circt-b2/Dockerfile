# CIRCT Vulnerability CVE-XXXX-XXXXX Reproduction Environment
# Vulnerability: Inconsistent Array Indexing Handling in Sensitivity Lists
# Affected Version: CIRCT firtool-1.139.0
# Issue: https://github.com/llvm/circt/issues/9469

FROM ghcr.io/m2kar/eda-vulns/circt-base:firtool-1.139.0

LABEL maintainer="M2kar <m2kar@github>"
LABEL description="Reproduction environment for CIRCT vulnerability - [Calyx] Segfault in SCFToCalyx when func.call exists inside loop body"
LABEL version="1.0"
LABEL circt.version="firtool-1.139.0"
LABEL vulnerability.id="CVE-PENDING"
LABEL vulnerability.cvss="5.3"

# Set working directory
WORKDIR /repro

# Copy test files
COPY bug.mlir /repro/
COPY reproduce.sh /repro/

# Make reproduction script executable
RUN chmod +x /repro/reproduce.sh

# Verify CIRCT installation
RUN circt-verilog --version && \
    echo "CIRCT installation successful"

# Create output directory for results
RUN mkdir -p /repro/output

# Default command: run full reproduction
#CMD ["/repro/reproduce.sh", "--all"]
CMD ["/bin/bash", "-c", "mlir-opt --lower-affine --scf-for-to-while bug.mlir | circt-opt --pass-pipeline='builtin.module(lower-scf-to-calyx{top-level-function=f})' 2>output/error.log"]
#CMD ["which","mlir-opt"]