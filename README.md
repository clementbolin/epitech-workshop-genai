# LLM Code Fixing Agent

A Go-based tool that uses LM Studio's local LLM to automatically detect and fix code errors.

## Prerequisites

- Go 1.20 or higher
- [LM Studio](https://lmstudio.ai) installed
- `llama-3.2-1b-instruct` model loaded in LM Studio

## Setup

1. Start LM Studio:
   - Load the `llama-3.2-1b-instruct` model
   - Start the server on port 1234
   - Enable "Just-in-time model loading"

2. Build the agent:
```bash
go build -o codefixer cmd/codefixer/main.go
```

## Usage

```bash
./codefixer <filename>
```

Example:
```bash
./codefixer test.go
```

## Code Structure

### Main Components

1. **Constants and Configuration**
```go
const (
    baseURL    = "http://localhost:1234/v1"
    modelName  = "llama-3.2-1b-instruct"
    apiTimeout = 120 * time.Second
)
```

2. **Request/Response Types**
```go
type CodeFixRequest struct {
    Model          string    `json:"model"`
    Messages       []Message `json:"messages"`
    Tools          []Tool    `json:"tools,omitempty"`
    ResponseFormat struct {
        Type       string     `json:"type"`
        JSONSchema JSONSchema `json:"json_schema,omitempty"`
    } `json:"response_format,omitempty"`
    Temperature float32 `json:"temperature"`
    Stream      bool    `json:"stream"`
}

type CodeFix struct {
    OriginalCode string `json:"original_code"`
    FixedCode    string `json:"fixed_code"`
    Explanation  string `json:"explanation"`
    Language     string `json:"language"`
    ErrorType    string `json:"error_type"`
}
```

3. **Core Functions**

```go
// Server health check
func checkServerAvailable() bool {
    client := &http.Client{Timeout: 5 * time.Second}
    resp, err := client.Get(baseURL + "/models")
    return err == nil && resp.StatusCode == http.StatusOK
}

// Code analysis and fixing
func analyzeAndFixCode(code string) (*CodeFix, error) {
    // Setup tools and schema
    // Send request to LLM
    // Parse and return response
}

// Safe file operations
func validateAndSave(originalFile string, fix *CodeFix) error {
    // Create backup
    // Write fixed code
    // Validate changes
}
```

## Implementation Details

### 1. JSON Schema Structure
```go
schema := JSONSchema{}
schema.Schema.Type = "object"
schema.Schema.Properties = map[string]Property{
    "fixed_code":    {Type: "string"},
    "explanation":   {Type: "string"},
    "language":      {Type: "string"},
    "error_type":    {Type: "string"},
}
```

### 2. Tool Definition
```go
tools := []Tool{
    {
        Type: "function",
        Function: Function{
            Name:        "perform_code_analysis",
            Description: "Analyzes code for errors and suggests fixes",
            Parameters: Parameters{
                Type: "object",
                Properties: map[string]Property{
                    "code": {
                        Type:        "string",
                        Description: "The code to analyze",
                    },
                    "language": {
                        Type:        "string",
                        Description: "Programming language of the code",
                        Enum:        []string{"go", "python", "javascript", "java", "c++"},
                    },
                },
                Required: []string{"code", "language"},
            },
        },
    },
}
```

## Safety Features

1. **Backup Creation**
   - Timestamped backups before modifications
   - Original file preserved as `filename.YYYYMMDDHHMMSS.bak`

2. **User Confirmation**
   - Interactive prompt before applying changes
   - Clear display of proposed fixes

3. **Code Validation**
   - Go code compilation check
   - Error reporting for invalid fixes

## Example Workflow

1. Create a test file with an error:
```go
package main

func main() {
    fmt.Println("Hello World"
}
```

2. Run the fixer:
```bash
./codefixer test.go
```

3. Review the output:
```
=== Code Fix Report ===
Language: go
Error Type: SyntaxError

Original Code:
[displays original code]

Fixed Code:
[displays fixed code]

Explanation:
[shows what was fixed and why]

Apply these changes? [y/N]:
```

4. Confirm the changes:
   - Type 'y' to apply
   - A backup is created
   - The file is updated
   - Go compilation is verified

## Troubleshooting

| Error | Solution |
|-------|----------|
| Server not available | Check if LM Studio is running on port 1234 |
| Invalid JSON schema | Verify schema structure in request |
| Permission denied | Check file write permissions |
| Compilation failed | Review the fixed code for errors |

## Resources

- [LM Studio Documentation](https://lmstudio.ai/docs)
- [Go Documentation](https://golang.org/doc/)

## Complete code

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"os/exec"
	"strings"
	"time"
)

const (
	baseURL    = "http://localhost:1234/v1"
	modelName  = "llama-3.2-1b-instruct"
	apiTimeout = 120 * time.Second
)

type CodeFixRequest struct {
	Model          string    `json:"model"`
	Messages       []Message `json:"messages"`
	Tools          []Tool    `json:"tools,omitempty"`
	ResponseFormat struct {
		Type       string     `json:"type"`
		JSONSchema JSONSchema `json:"json_schema,omitempty"`
	} `json:"response_format,omitempty"`
	Temperature float32 `json:"temperature"`
	Stream      bool    `json:"stream"`
}

type JSONSchema struct {
	Schema struct {
		Type       string              `json:"type"`
		Properties map[string]Property `json:"properties"`
		Required   []string            `json:"required"`
	} `json:"schema"`
}

type Property struct {
	Type        string   `json:"type"`
	Description string   `json:"description,omitempty"`
	Enum        []string `json:"enum,omitempty"`
}

type Message struct {
	Role      string     `json:"role"`
	Content   string     `json:"content"`
	ToolCalls []ToolCall `json:"tool_calls,omitempty"`
}

type ToolCall struct {
	ID       string       `json:"id"`
	Type     string       `json:"type"`
	Function FunctionCall `json:"function"`
}

type FunctionCall struct {
	Name      string `json:"name"`
	Arguments string `json:"arguments"`
}

type Tool struct {
	Type     string   `json:"type"`
	Function Function `json:"function"`
}

type Function struct {
	Name        string     `json:"name"`
	Description string     `json:"description"`
	Parameters  Parameters `json:"parameters"`
}

type Parameters struct {
	Type       string              `json:"type"`
	Properties map[string]Property `json:"properties"`
	Required   []string            `json:"required"`
}

type CodeFixResponse struct {
	ID      string `json:"id"`
	Choices []struct {
		Message struct {
			Content   string     `json:"content"`
			ToolCalls []ToolCall `json:"tool_calls"`
		} `json:"message"`
	} `json:"choices"`
}

type CodeFix struct {
	OriginalCode string `json:"original_code"`
	FixedCode    string `json:"fixed_code"`
	Explanation  string `json:"explanation"`
	Language     string `json:"language"`
	ErrorType    string `json:"error_type"`
}

func main() {
	if !checkServerAvailable() {
		fmt.Println("LM Studio server not available. Please ensure it's running at", baseURL)
		os.Exit(1)
	}

	if len(os.Args) < 2 {
		fmt.Println("Usage: codefixer <filename>")
		os.Exit(1)
	}

	filename := os.Args[1]
	content, err := os.ReadFile(filename)
	if err != nil {
		fmt.Printf("Error reading file: %v\n", err)
		os.Exit(1)
	}

	fix, err := analyzeAndFixCode(string(content))
	if err != nil {
		fmt.Printf("Error fixing code: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("\n=== Code Fix Report ===")
	fmt.Printf("Language: %s\nError Type: %s\n", fix.Language, fix.ErrorType)
	fmt.Println("\nOriginal Code:")
	fmt.Println(fix.OriginalCode)
	fmt.Println("\nFixed Code:")
	fmt.Println(fix.FixedCode)
	fmt.Println("\nExplanation:")
	fmt.Println(fix.Explanation)

	if err := validateAndSave(filename, fix); err != nil {
		fmt.Printf("\nError: %v\n", err)
	} else {
		fmt.Println("\nUpdate successful!")
	}
}

func analyzeAndFixCode(code string) (*CodeFix, error) {
	tools := []Tool{
		{
			Type: "function",
			Function: Function{
				Name:        "perform_code_analysis",
				Description: "Analyzes code for errors and suggests fixes",
				Parameters: Parameters{
					Type: "object",
					Properties: map[string]Property{
						"code": {
							Type:        "string",
							Description: "The code to analyze",
						},
						"language": {
							Type:        "string",
							Description: "Programming language of the code",
							Enum:        []string{"go", "python", "javascript", "java", "c++"},
						},
					},
					Required: []string{"code", "language"},
				},
			},
		},
	}

	schema := JSONSchema{}
	schema.Schema.Type = "object"
	schema.Schema.Properties = map[string]Property{
		"original_code": {Type: "string"},
		"fixed_code":    {Type: "string"},
		"explanation":   {Type: "string"},
		"language":      {Type: "string"},
		"error_type":    {Type: "string"},
	}
	schema.Schema.Required = []string{"fixed_code", "explanation", "language", "error_type"}

	messages := []Message{
		{
			Role: "system",
			Content: "You are an expert code debugging assistant. Analyze the provided code, " +
				"identify errors, and provide a corrected version with explanations. " +
				"Use structured JSON output format.",
		},
		{
			Role:    "user",
			Content: fmt.Sprintf("Analyze and fix this code:\n```\n%s\n```", code),
		},
	}

	request := CodeFixRequest{
		Model:    modelName,
		Messages: messages,
		Tools:    tools,
		ResponseFormat: struct {
			Type       string     `json:"type"`
			JSONSchema JSONSchema `json:"json_schema,omitempty"`
		}{
			Type:       "json_schema",
			JSONSchema: schema,
		},
		Temperature: 0.3,
		Stream:      false,
	}

	response, err := sendChatRequest(request)
	if err != nil {
		return nil, err
	}

	var codeFix CodeFix
	if err := json.Unmarshal([]byte(response.Choices[0].Message.Content), &codeFix); err != nil {
		return nil, fmt.Errorf("error parsing JSON response: %v", err)
	}

	codeFix.OriginalCode = code
	return &codeFix, nil
}

func sendChatRequest(request CodeFixRequest) (*CodeFixResponse, error) {
	client := &http.Client{Timeout: apiTimeout}

	requestBody, err := json.Marshal(request)
	if err != nil {
		return nil, err
	}

	req, err := http.NewRequest("POST", baseURL+"/chat/completions", bytes.NewBuffer(requestBody))
	if err != nil {
		return nil, err
	}
	req.Header.Set("Content-Type", "application/json")

	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(resp.Body)
		return nil, fmt.Errorf("API request failed: %s\nResponse: %s", resp.Status, body)
	}

	var response CodeFixResponse
	if err := json.NewDecoder(resp.Body).Decode(&response); err != nil {
		return nil, err
	}

	return &response, nil
}

func checkServerAvailable() bool {
	client := &http.Client{Timeout: 5 * time.Second}
	resp, err := client.Get(baseURL + "/models")
	if err != nil {
		return false
	}
	defer resp.Body.Close()
	return resp.StatusCode == http.StatusOK
}

func validateAndSave(originalFile string, fix *CodeFix) error {
	fmt.Print("\nApply these changes? [y/N]: ")

	tty, err := os.Open("/dev/tty")
	if err != nil {
		return fmt.Errorf("error opening terminal: %v", err)
	}
	defer tty.Close()

	var confirm string
	_, err = fmt.Fscanln(tty, &confirm)
	if err != nil && err != io.EOF {
		return fmt.Errorf("error reading input: %v", err)
	}

	if strings.ToLower(confirm) != "y" {
		return fmt.Errorf("user cancelled the operation")
	}

	// Create backup with timestamp
	backupName := fmt.Sprintf("%s.%s.bak", originalFile, time.Now().Format("20060102150405"))
	if err := os.Rename(originalFile, backupName); err != nil {
		return fmt.Errorf("error creating backup: %v", err)
	}

	// Write fixed content to original filename
	if err := os.WriteFile(originalFile, []byte(fix.FixedCode), 0644); err != nil {
		return fmt.Errorf("error writing fixed code: %v", err)
	}

	fmt.Printf("\nBackup saved to %s\n", backupName)

	// Validate the fixed code
	if fix.Language == "go" && strings.HasSuffix(originalFile, ".go") {
		fmt.Println("\nValidating Go code...")
		cmd := exec.Command("go", "build", "-o", "/dev/null", originalFile)
		if output, err := cmd.CombinedOutput(); err != nil {
			return fmt.Errorf("build failed: %s\n%s", err, string(output))
		}
		fmt.Println("Code compiled successfully!")
	}

	return nil
}
```
