rest.http

###
POST http://192.168.0.200/openai/v1/chat/completions
Content-Type: application/json
Host: qwen-llm.kserve-test.didc.local

{
  "model": "qwen",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant that provides clear and concise answers."
    },
    {
      "role": "user",
      "content": "이란과 북한의 국제 관계는 어떠한지?"
    }
  ],
  "max_tokens": 150,
  "temperature": 0.7,
  "stream": false
}

###

POST http://192.168.0.200/v1/models/sklearn-iris:predict
Host: sklearn-iris.kserve-test.didc.local
Content-Type: application/json

{
  "instances": [
    [2.8,  5.8,  2.8,  1.4],
    [1.0,  3.4,  4.5,  1.6]
  ]
}
