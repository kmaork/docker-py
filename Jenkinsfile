#!groovy

def imageNameBase = "dockerbuildbot/docker-py"
def imageNamePy2
def imageNamePy3
def images = [:]

// Note: Swarm in dind seem notoriously flimsy with 1.12.1+, which is why we're
// sticking with 1.12.0 for the 1.12 series
def dockerVersions = ["1.12.0", "1.13.0"]

def buildImage = { name, buildargs, pyTag ->
  img = docker.image(name)
  try {
    img.pull()
  } catch (Exception exc) {
    img = docker.build(name, buildargs)
    img.push()
  }
  images[pyTag] = img.id
}

def buildImages = { ->
  wrappedNode(label: "ubuntu && !zfs && amd64", cleanWorkspace: true) {
    stage("build image") {
      checkout(scm)

      imageNamePy2 = "${imageNameBase}:py2-${gitCommit()}"
      imageNamePy3 = "${imageNameBase}:py3-${gitCommit()}"

      buildImage(imageNamePy2, ".", "py2.7")
      buildImage(imageNamePy3, "-f Dockerfile-py3 .", "py3.5")
    }
  }
}

def runTests = { Map settings ->
  def dockerVersion = settings.get("dockerVersion", null)
  def pythonVersion = settings.get("pythonVersion", null)
  def testImage = settings.get("testImage", null)

  if (!testImage) {
    throw new Exception("Need test image object, e.g.: `runTests(testImage: img)`")
  }
  if (!dockerVersion) {
    throw new Exception("Need Docker version to test, e.g.: `runTests(dockerVersion: '1.12.3')`")
  }
  if (!pythonVersion) {
    throw new Exception("Need Python version being tested, e.g.: `runTests(pythonVersion: 'py2.7')`")
  }

  { ->
    wrappedNode(label: "ubuntu && !zfs && amd64", cleanWorkspace: true) {
      stage("test python=${pythonVersion} / docker=${dockerVersion}") {
        checkout(scm)
        def dindContainerName = "dpy-dind-\$BUILD_NUMBER-\$EXECUTOR_NUMBER"
        def testContainerName = "dpy-tests-\$BUILD_NUMBER-\$EXECUTOR_NUMBER"
        try {
          sh """docker run -d --name  ${dindContainerName} -v /tmp --privileged \\
            dockerswarm/dind:${dockerVersion} docker daemon -H tcp://0.0.0.0:2375
          """
          sh """docker run \\
            --name ${testContainerName} --volumes-from ${dindContainerName} \\
            -e 'DOCKER_HOST=tcp://docker:2375' \\
            --link=${dindContainerName}:docker \\
            ${testImage} \\
            py.test -v -rxs tests/integration
          """
        } finally {
          sh """
            docker stop ${dindContainerName} ${testContainerName}
            docker rm -vf ${dindContainerName} ${testContainerName}
          """
        }
      }
    }
  }
}


buildImages()

def testMatrix = [failFast: false]

for (imgKey in new ArrayList(images.keySet())) {
  for (version in dockerVersions) {
    testMatrix["${imgKey}_${version}"] = runTests([testImage: images[imgKey], dockerVersion: version, pythonVersion: imgKey])
  }
}

parallel(testMatrix)