curl -X 'POST' \
  'https://localhost:7192/api/OnBoarding/Insert_UserApplication_WithFiles' \
  -H 'accept: text/plain' \
  -H 'Content-Type: application/json' \
  -d '{
  "baseRequest": {
    "correlationId": "1b2b57f6-f5c3-451b-988a-3097cec15a02"
  },
  "correlationId": "1b2b57f6-f5c3-451b-988a-3097cec15a03",
  "external_Id": "15281520",
  "app_FilesList": [
    {
      "file_Name": "Document.pdf",
      "file_Type": "application/pdf",
      "file_Size": 1024,
      "file_Data": 0x255044462D312E342025E2E3CFD30A
    }
  ]
}'


{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "request": [
      "The request field is required."
    ],
    "$.app_FilesList[0].file_Data": [
      "'x' is an invalid end of a number. Expected a delimiter. Path: $.app_FilesList[0].file_Data | LineNumber: 11 | BytePositionInLine: 20."
    ]
  },
  "traceId": "00-fda24d324cbac689341a7b9a4329eb9e-bbc09ef05d141756-00"
}
