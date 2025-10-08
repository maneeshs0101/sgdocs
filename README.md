AsyncHttpClient client = new DefaultAsyncHttpClient();
client.prepare("POST", "https://api.sgdocs.dev.euw.gbis.sg-azure.com/api/v3/workspaces/2467dfb1-b9d2-4236-bda0-1682d6263f2b/documents/e0779bf4-1be8-46c6-a4a5-c61c218def99/contents")
  .setHeader("authorization", "Bearer token")
  .setHeader("content-type", "application/pdf")
  .setHeader("content-length", "154158")
  .setHeader("content-disposition", "binary;filename=A407925050188327.pdf")
  .setBody("C:\\Users\\mshivapr022725\\OneDrive - GROUP DIGITAL WORKPLACE\\Documents\\A407925050188327.pdf")
  .execute()
  .toCompletableFuture()
  .thenAccept(System.out::println)
  .join();

client.close();
