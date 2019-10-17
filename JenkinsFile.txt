stage 'CI'
node {

	checkout scm
	
    //git branch: 'jenkins2-course', 
    //    url: 'https://github.com/g0t4/solitaire-systemjs-course'

    // pull dependencies from npm
    // on windows use: bat 'npm install'
    bat 'npm install'

    // stash code & dependencies to expedite subsequent testing
    // and ensure same code & dependencies are used throughout the pipeline
    // stash is a temporary archive
    stash name: 'everything', 
          excludes: 'test-results/**', 
          includes: '**'
    
    // test with PhantomJS for "fast" "generic" results
    // on windows use: bat 'npm run test-single-run -- --browsers PhantomJS'
    bat 'npm run test-single-run -- --browsers PhantomJS'
    
    // archive karma test results (karma is configured to export junit xml files)
    step([$class: 'JUnitResultArchiver', 
          testResults: 'test-results/**/test-results.xml'])
          
    stage 'archiving'
    archiveArtifacts 'app/*'
}

node ('windows-agent') {
    bat 'dir'
    bat 'DEL /F/Q/S *.*'
    unstash 'everything'
    bat 'dir'
}

//parallel integration testing
stage 'Browser Testing'
parallel chrome: {
    runTests("Chrome")
}, firefox: {
    runTests("Firefox")
}

/*  DISABLED FOR NOW
node {
    notify('Deploy to staging?')    
}
*/

input 'Deploy to staging?'

//limit concurrency so we don't perform simulaneous deploys
//and if multiple pipelines areexecuting,
//only the newest will be allowed through, the rest will be canceled
stage name: "Deploy", concurrency: 1
/* DISABLED FOR NOW
node {
    //write buildnumber to index page so we can see this update
    bat "echo '<h1>${env.BUILD_DISPLAY_NAME}</h1>' >> app/index.html"
    
    //deploy to a docker container mapped to port xxxx
    bat 'docker-compose up -d --build'
    
    notify('solitaire dployed')
}
*/

def runTests(browser){
    node{
        bat 'DEL /F/Q/S *.*'
        unstash 'everything'
        bat "npm run test-single-run -- --browsers ${browser}"
        // archive karma test results (karma is configured to export junit xml files)
        step([$class: 'JUnitResultArchiver', 
          testResults: 'test-results/**/test-results.xml'])
    }
}

def notify(status){
    emailext (
      to: "jgd@it-minds.dk",
      subject: "${status}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
      body: """<p>${status}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
        <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
    )
}