# 📧 Cold Mail Generator
Cold email generator for services company using groq, langchain and streamlit. It allows users to input the URL of a company's careers page. The tool then extracts job listings from that page and generates personalized cold emails. These emails include relevant portfolio links sourced from a vector database, based on the specific job descriptions. 

**Imagine a scenario:**

- Nike needs a Principal Software Engineer and is spending time and resources in the hiring process, on boarding, training etc
- Atliq is Software Development company can provide a dedicated software development engineer to Nike. So, the business development executive (Mohan) from Atliq is going to reach out to Nike via a cold email.

![img.png](imgs/img.png)

## Architecture Diagram
![img.png](imgs/architecture.png)

## Set-up
0. RM: Use virtual env.
venv\Scripts\activate
pip install streamlit
pip install langchain-groq

trychroma.com for Chroma VectorDB
    - pip install chromadb 

1. To get started we first need to get an API_KEY from here: https://console.groq.com/keys. Inside `app/.env` update the value of `GROQ_API_KEY` with the API_KEY you created. 


2. To get started, first install the dependencies using:
    ```commandline
     pip install -r requirements.txt
    ```
   
3. Run the streamlit app:
   ```commandline
   streamlit run app/main.py
   ```
   

Copyright (C) Codebasics Inc. All rights reserved.

**Additional Terms:**
This software is licensed under the MIT License. However, commercial use of this software is strictly prohibited without prior written permission from the author. Attribution must be given in all copies or substantial portions of the software.

RM Notes:

(1) Read the URL for Nike open jobs
		- Using loader = WebBaseLoader("https://careers.nike.com...")
		- page_date = loader.page_content
		
		
	(2)Tell the LLM to extract the key data from this website page_data, return as JSON
		(2.1) First prepare a prompt for the LLM, including page_data 
			prompt_extract = PromptTemplate.from_template(
				"""
				### SCRAPED TEXT FROM WEBSITE:
				{page_data}
				### INSTRUCTION:
				The scraped text is from the career's page of a website...
				### VALID JSON (NO PREAMBLE):   
		
		(2.2) Create the chain object
			chain_extract = prompt_extract | llm 
			
		(2.3) Call the Groq LLM, putting in the page_data value into {page_data} above
			res = chain_extract.invoke(input={'page_data':page_data})
			
	The LLM gives us a nice JSON containing the details of the job:
	json
{
  "role": "Accounting/Administrative Assistant",
  "experience": "1 year of retail or consumer service experience",
  "skills": [
    "Proficient knowledge of office practices, procedures, and equipment",
    "Intermediate skills in Microsoft Office products including Word and Excel",
    "Strong customer service skills",
    "Ability to use the Internet/Intranet as a resource for department work activities",
    "Strong attention to detail and deadlines"
  ],
  "description": "As a Nike Retail Accounting Admin, you..."
}
		(2.4) Convert this string to a JSON object (dictionary)
			JsonOutputParser()
			job = json_parser.parse(res.content)  #Now has the above keys from #2.3 e.g. job["skills"] is the array of skills :
				['Proficient knowledge of office practices, procedures, and equipment',
					'Intermediate skills in Microsoft Office products including Word and Excel',
					'Strong customer service skills',
					'Ability to use the Internet/Intranet as a resource for department work activities',
					'Strong attention to detail and deadlines']
					
		These are the skills required for the Job!
		
	(3) Now get our capabilities from our Portfolio
	
		(3.1) Read the portfolio text file
			import pandas as pd
			df = pd.read_csv("my_portfolio.csv")  #data-frame
			
			Which gives
				Techstack	Links
			0	React, Node.js, MongoDB	https://example.com/react-portfolio
			1	Angular,.NET, SQL Server	https://example.com/angular-portfolio
			etc.
			
		(3.2) Read each df row into the VectorDB. Store the Techstack column as "documents" and Links as "metadatas".
				client = chromadb.PersistentClient('vectorstore')
				collection = client.get_or_create_collection(name="portfolio")

				if not collection.count():
					for _, row in df.iterrows():
						collection.add(documents=row["Techstack"],
							metadatas={"links": row["Links"]},
							ids=[str(uuid.uuid4())])

	(4) Now key step: Query the Vector DB for all the job skills from Nike.
		This will do a _semantic_ search in the VectorDB and find rows from my Portfolio, which match the job opening!
			links = collection.query(query_texts=job['skills'], n_results=2).get('metadatas', [])
		We thus get the links [showing our competence for the precise skills asked for by Nike] which we need to insert in our final email.
		
	(5) So now the final email:
			Prompt the LLM with the full #job [required skills] plus the ones which match from our Portfolio [#links]
	prompt_email = PromptTemplate.from_template(
        """
        ### JOB DESCRIPTION:
        {job_description}
        
        ### INSTRUCTION:
        You are Mohan, a business development executive at AtliQ. AtliQ is an AI & Software Consulting company dedicated to facilitating
        the seamless integration of business processes through automated tools. 
        Over our experience, we have empowered numerous enterprises with tailored solutions, fostering scalability, 
        process optimization, cost reduction, and heightened overall efficiency. 
        Your job is to write a cold email to the client regarding the job mentioned above describing the capability of AtliQ 
        in fulfilling their needs.
        Also add the most relevant ones from the following links to showcase Atliq's portfolio: {link_list}
        Remember you are Mohan, BDE at AtliQ. 
        Do not provide a preamble.
        ### EMAIL (NO PREAMBLE):
        
        """
        )

	chain_email = prompt_email | llm
	res = chain_email.invoke({"job_description": str(job), "link_list": links})
	print(res.content)
	
	Done!
	
	Subject: Expert Solution for Streamlining Retail Operations

Dear Hiring Manager,

I came across the job description for an Accounting/Administrative Assistant at Nike, and I understood the requirements for the role. As a Business Development Executive at AtliQ, an AI & Software Consulting company, I'd like to introduce you to our expertise in automating business processes and optimizing operations.

The job description highlights the need for efficient cashiering tasks, scheduling support, and troubleshooting. AtliQ can help you achieve these goals by implementing tailored software solutions. Our expertise includes developing customized tools for automating tasks, enhancing customer service, and improving overall efficiency.

Some of our notable projects showcase our capabilities in this domain. For instance, you can explore our portfolio in automation and process optimization through our DevOps portfolio: https://example.com/devops-portfolio. Additionally, our expertise in developing customized solutions using Python can be seen in our Machine Learning portfolio: https://example.com/ml-python-portfolio.

By partnering with AtliQ, you can expect:

* Scalability in your retail operations
* Process optimization for tasks like cashiering and scheduling
* Cost reduction through automation
* Heightened overall efficiency

I'd be delighted to discuss how AtliQ's expertise can support your business needs. Please feel free to reply to this email or schedule a call at your convenience.

Looking forward to exploring how we can contribute to your success.

Best regards,

Mohan
Business Development Executive
AtliQ