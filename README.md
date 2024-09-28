package main

import ( "bufio, "flag", "fmt","io","os","runtime","strings", "sync"}

)

func main() {
 pattern := flag.String("pattern", "", "Pattern to search for")
 numWorkers := flag.Int("workers", runtime.NumCPU(), "Number of worker goroutines")
 flag.Parse()

 if *pattern == "" {
  fmt.Println("Error: Pattern must be provided. Use -pattern flag.")
  os.Exit(1)
 }

 files := flag.Args()
 if len(files) == 0 {
  fmt.Println("Error: No files provided. Use filenames as arguments.")
  os.Exit(1)
 }

 // Create a wait group to track the completion of worker goroutines
 var wg sync.WaitGroup
 wg.Add(len(files))

 // Create a channel to send results from workers
 results := make(chan string)

 // Start worker goroutines
 for i := 0; i < *numWorkers; i++ {
  go func() {
   defer wg.Done()
   for file := range files {
    // Read file line by line
    f, err := os.Open(file)
    if err != nil {
     fmt.Println("Error opening file:", file, err)
     continue
    }
    defer f.Close()

    scanner := bufio.NewScanner(f)
    for scanner.Scan() {
     line := scanner.Text()
     if strings.Contains(line, *pattern) {
      results <- fmt.Sprintf("%s:%s", file, line)
     }
    }
    if err := scanner.Err(); err != nil {
     fmt.Println("Error reading file:", file, err)
     continue
    }
   }
  }
 }
 // Send files to worker goroutines
 for _, file := range files {
  files <- file
 }
 close(files)

 // Collect results from workers
 go func() {
  defer close(results)
  wg.Wait()
 }

 // Print the results
 for result := range results {
  fmt.Println(result)
 }
}
