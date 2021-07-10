*WARNING:* Kythe is alpha.  Use at your own risk.

Kythe is a pluggable, (mostly) language-agnostic ecosystem for building tools
that work with code.  This release contains the core set of indexers,
extractors, and tools directly supported by the Kythe team.

*License:* Apache Licence, Version 2.0

# Contents
 - indexers
   - cxx_indexer              :: C++ indexer
   - java_indexer.jar         :: Java indexer
 - extractors
   - bazel_cxx_extractor      :: C++ extractor for Bazel extra_actions
   - bazel_java_extractor.jar :: Java extractor for Bazel extra_actions
   - cxx_extractor            :: C++ extractor
   - javac_extractor.jar      :: Java extractor
   - javac-wrapper.sh         :: javac wrapper script for extractor
 - proto                      :: Protocol buffer definitions of public APIs
 - tools
   - dedup_stream             :: Removes duplicates entries from a delimited stream
   - directory_indexer        :: Emits Kythe file nodes for some local paths
   - entrystream              :: Generic Kythe entry stream processor
   - http_server              :: HTTP/GRPC server for Kythe service APIs (xrefs, filetree, search)
   - indexpack                :: Converts between .kindex archives and indexpacks
   - kwazthis                 :: Determine what Kythe node(s) are at a particular point in a file
   - kythe                    :: CLI for the service APIs exposed by http_server
   - read_entries             :: Dumps a GraphStore's contents as an entry stream
   - triples                  :: Converts an entry stream (or GraphStore) to N-Triples
   - verifier                 :: Verifies indexer outputs with source-inlined goals
   - viewindex                :: Dumps a .kindex file as JSON
   - write_entries            :: Writes an entry stream to a GraphStore
   - write_tables             :: Processes a GraphStore into efficient serving tables for http_server

# Dependencies
 - Java JDK >=8
 - libncurses
 - libuuid

## Debian Jessie Install

    echo "deb http://http.debian.net/debian jessie-backports main" >> /etc/apt/sources.list
    apt-get install openjdk-8-jdk libncurses5 libssl1.0.0

# End-to-end Java Example

```
# Install Kythe
tar xzf kythe-v*.tar.gz
rm -rf /opt/kythe
mv kythe-v*/ /opt/kythe

git clone https://github.com/GoogleCloudPlatform/DataflowJavaSDK.git
cd DataflowJavaSDK

# Setup the Java extractor's environment
export REAL_JAVAC="$(which javac)"
export JAVAC_EXTRACTOR_JAR=/opt/kythe/extractors/javac_extractor.jar
export KYTHE_ROOT_DIRECTORY="$PWD"
export KYTHE_OUTPUT_DIRECTORY="/tmp/kythe"
export KYTHE_CORPUS=DataflowJavaSDK
mkdir -p "$KYTHE_OUTPUT_DIRECTORY"

# Force Maven to compile with extractors/javac-wrapper.sh
mvn clean compile \
  -Dmaven.compiler.forceJavacCompilerUse=true \
  -Dmaven.compiler.fork=true \
  -Dmaven.compiler.executable=/opt/kythe/extractors/javac-wrapper.sh

cd ..

# Index the resulting .kindex files, deduplicate the entries, and finally store
# them in a GraphStore
GRAPHSTORE=/tmp/kythe_graphstore
rm -rf "$GRAPHSTORE"
find "$KYTHE_OUTPUT_DIRECTORY" -name '*.kindex' | \
  xargs -L1 java -jar /opt/kythe/indexers/java_indexer.jar | \
  /opt/kythe/tools/dedup_stream | \
  /opt/kythe/tools/write_entries --graphstore $GRAPHSTORE

# Process the GraphStore into serving tables
SERVING=/tmp/kythe_serving
rm -rf "$SERVING"
/opt/kythe/tools/write_tables --graphstore $GRAPHSTORE --out "$SERVING"

# Launch Kythe's service APIs as an HTTP server listening to only local
# connections on port 9898.  Using `--listen :9898` instead will allow
# connections from other networked machines.
/opt/kythe/tools/http_server --serving_table "$SERVING" \
  --public_resources /opt/kythe/web/ui --listen localhost:9898

# The sample web UI will be serving at http://localhost:9898

# /opt/kythe/tools/kythe and /opt/kythe/tools/kwazthis can be used with the
# local running server by passing the '--api=http://localhost:9898' flag.
```

# Usage

## Extractors

`extractors/cxx_extractor` and `extractors/javac_extractor.jar` are
flag-compatible replacements for clang and javac, respectively.  Instead of
performing a full compilation, they extract the compilation's full context
including all file inputs and archive them in a .kindex file.

Since each extractor is meant to replace a compiler, all configuration is passed
by the following environment variables:

    KYTHE_ROOT_DIRECTORY (required): root directory of the repository being compiled.
        This helps the extractor correctly infer file input paths.
    KYTHE_OUTPUT_DIRECTORY (required): output directory for resulting .kindex
        files
    KYTHE_VNAMES: path to a JSON-encoded VNames configuration file.  See
        https://godoc.org/github.com/google/kythe/kythe/go/storage/vnameutil for
        more details on the file's format and
        https://kythe.io/repo/kythe/data/vnames.json for an example.
    KYTHE_CORPUS: the name of the corpus for all constructed VNames (only used if
        KYTHE_VNAMES is unset; defaults to "kythe")

`extractors/javac-wrapper.sh` is provided as one way to inject an extractor into
a build system -- by wrapping each compiler command.  For instance, Maven
compilations can easily be extracted by replacing `$JAVA_HOME/bin/javac` with
this script and setting the following Maven options:

    -Dmaven.compiler.forceJavacCompilerUse=true
    -Dmaven.compiler.fork=true
    -Dmaven.compiler.executable=$JAVA_HOME/bin/javac

Read `extractors/javac-wrapper.sh` for more details on its usage and see
examples usages in the Docker images defined at
https://kythe.io/repo/kythe/java/com/google/devtools/kythe/extractors/java/standalone/Dockerfile
and
https://kythe.io/repo/kythe/java/com/google/devtools/kythe/extractors/java/maven/Dockerfile.

### Examples:

    export KYTHE_ROOT_DIRECTORY="$PWD"
    export KYTHE_OUTPUT_DIRECTORY=/tmp/kythe

    mkdir -p "$KYTHE_OUTPUT_DIRECTORY"

    java -Xbootclasspath/p:extractors/javac_extractor.jar \
      -jar extractors/javac_extractor.jar \
      -cp "${$(find third_party -name '*.jar' -printf '%p:')%:}" \
      $(find src/main -name '*.java')

    extractors/cxx_extractor -Ithird_party/include some.cc

## Indexers

`indexers/cxx_indexer` and `indexers/java_indexer.jar` analyze the .kindex files
produced by the extractors and emit a stream of protobuf wire-encoded facts
(entries) that conform to https://kythe.io/schema.  The output stream can be
processed by many of the accompanying binaries in the tools/ directory.

### Examples

    java -jar indexers/java_indexer.jar \
      /tmp/kythe/065c7a9a0789d09b73d74b0ca17cfcec6643d69a6218d095d19a770b10dffdf9.kindex \
      > java.entries

    indexers/cxx_indexers \
      /tmp/kythe/579d266e5914257a9bd4458eb9b218690280ae15123d642025f224d10f64e6f3.kindex \
      > cxx.entries