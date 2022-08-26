# pttgc-devops-lab07
This module will provide hands-on experience to attendee to create the DevOps workflows for trunk based strategy.

## Exercise 1 - Pull Request Workflow

- Create new repository `devops-lab07` and copy `src` folder, `Dockerfile` to your repository
- Create new workflow `.github/workflows/pull-request-workflow.yaml`
```yaml
name: Pull Request workflow

on: 
  pull_request:
    types:
      - opened

jobs:
  unittest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-go@v3
        with:
          go-version-file: './src/go.mod'
      - run: go mod tidy
        working-directory: ./src
      - run: go test ./...
        working-directory: ./src
  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-go@v3
        with:
          go-version-file: './src/go.mod'
      - run: go mod tidy
        working-directory: ./src
        name: Download libraries
      - run: go test -coverprofile coverage.out ./...
        working-directory: ./src
        name: Code Coverage Checking
      - run: go tool cover -func coverage.out -o coverage.txt
        working-directory: ./src
        name: Convert Code Coverage Report format
      - uses: actions/upload-artifact@v3
        name: Store code coverage report 
        with:
          name: code-coverage-report
          path: src/coverage.txt
  
  code-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Trivy vulnerability scanner in repo mode
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          ignore-unfixed: true
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL'
      - uses: actions/upload-artifact@v3
        with:
          name: code-scan-result
          path: trivy-results.sarif

  quality-gate:
    runs-on: ubuntu-latest
    needs: ["coverage", "unittest", "code-scan"]
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: code-coverage-report
          path: ./src
      - run: python coverage.py 1
        working-directory: ./src
        name: Simple Quality Gate

  comment:
    runs-on: ubuntu-latest
    needs: ["quality-gate"]
    steps:
      - uses: actions/script@v3
        with:
          script: |
            github.issues.createComment({
              issue_number: context.payload.pull_request.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: 'All Quality Gates is passed'
            })
```

- Commit and Push the code

- Create new branch, for example `test-pr`

- Ensure that you are in `test-pr` branch and create simple text file in the repository (any file will do)

- Create Pull Request 

- Go to `Actions` tab to see the workflow is running

- Merge Pull request to `main` branch and delete `test-pr` branch


## Exercise 2 - Merge Pull Request Workflow

- Using the same repository from exercise 1

- Add new workflow name `.github/workflows/merge-pr-workflow.yaml`
```yaml
name: Merge Pull Request Workflow

on: 
  pull_request:
    types:
      - closed

jobs:
  build-container:
    runs-on: ubuntu-latest
    if: github.event.pull_request.merged == true
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Build an image from Dockerfile
        run: |
          docker build . --file Dockerfile --tag pttgc-devops-lab07/blog-service:${{ github.sha }}

  deploy-dev:
    runs-on: ubuntu-latest
    needs: "build-container"
    steps:
      - name: Mock deployment step 
        run: echo 'Mock deploy to dev'
```

- Commit and push the code

- Create another new branch, e.g. `mr1`

- Ensure that you're in `mr1` branch, add simple text file to repository, then commit and push

- Create Pull Request

- Merge Pull Request

- Go to `Actions` tab to see your workflow is running