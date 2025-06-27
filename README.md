using System;
using System.Collections.Generic;
using System.Globalization;
using System.Media;
using System.Text.RegularExpressions;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;

namespace CybersecurityChatbotWPF
{
    #region Models
    public class TaskItem
    {
        public string Title { get; set; }
        public DateTime? ReminderDate { get; set; }
        public bool IsCompleted { get; set; }
    }

    public class QuizQuestion
    {
        public string Question { get; set; }
        public List<string> Choices { get; set; } = new();
        public string CorrectAnswer { get; set; }
        public string Explanation { get; set; }
        public bool IsTrueFalse { get; set; } = false;
    }
    #endregion

    public partial class MainWindow : Window
    {
        // Task & Activity log
        private readonly List<TaskItem> tasks = new();
        private readonly List<string> actionLog = new();
        private const int LogDisplayLimit = 10;

        // Quiz state
        private readonly List<QuizQuestion> quizQuestions = new();
        private int currentQuizIndex = -1;
        private int quizScore = 0;
        private bool quizActive = false;

        // Conversation flow states
        private bool awaitingGreeting = true;
        private bool awaitingName = false;
        private bool awaitingFavoriteTopic = false;
        private bool awaitingFeeling = false;

        private string userName = "";
        private string favoriteTopic = "";

        public MainWindow()
        {
            InitializeComponent();
            try
            {
                PlayVoiceGreeting("Assets/quiz_greeting.wav"); // Ensure this exists or comment out
            }
            catch { }
            DisplayAsciiArt();

            InitializeQuizQuestions();

            AppendChat("Bot: Say 'hey' to start chatting with me!");
            awaitingGreeting = true;

            RefreshTaskList();
            RefreshActivityLog();
        }

        #region Voice + ASCII

        private void PlayVoiceGreeting(string filePath)
        {
            try
            {
                SoundPlayer player = new(filePath);
                player.Play();
            }
            catch (Exception ex)
            {
                MessageBox.Show("Voice greeting failed to play: " + ex.Message);
            }
        }

        private void DisplayAsciiArt()
        {
            string asciiArt = "\n  ██████╗ ██╗   ██╗██████╗ ███████╗██████╗ ███████╗ ██████╗ ██╗   ██╗" +
                             "\n ██╔═══██╗██║   ██║██╔══██╗██╔════╝██╔══██╗██╔════╝██╔═══██╗╚██╗ ██╔╝" +
                             "\n ██║   ██║██║   ██║██████╔╝█████╗  ██████╔╝█████╗  ██║   ██║ ╚████╔╝ " +
                             "\n ██║▄▄ ██║██║   ██║██╔══██╗██╔══╝  ██╔═══╝ ██╔══╝  ██║   ██║  ╚██╔╝  " +
                             "\n ╚██████╔╝╚██████╔╝██║  ██║███████╗██║     ███████╗╚██████╔╝   ██║   " +
                             "\n  ╚══▀▀═╝  ╚═════╝ ╚═╝  ╚═╝╚══════╝╚═╝     ╚══════╝ ╚═════╝    ╚═╝   " +
                             "\n   Welcome to the Cybersecurity Chatbot!";
            MessageBox.Show(asciiArt, "Welcome", MessageBoxButton.OK, MessageBoxImage.Information);
        }

        #endregion

        #region Chat NLP Entry Point

        private void SendButton_Click(object sender, RoutedEventArgs e)
        {
            string input = UserInputBox.Text.Trim();
            if (string.IsNullOrEmpty(input)) return;

            AppendChat($"You: {input}");
            ProcessUserInput(input);
            UserInputBox.Text = string.Empty;
        }

        private void UserInputBox_KeyDown(object sender, KeyEventArgs e)
        {
            if (e.Key == Key.Enter)
            {
                SendButton_Click(sender, e);
                e.Handled = true;
            }
        }

        private void AppendChat(string message)
        {
            ChatHistory.Items.Add(message);
            ChatHistory.ScrollIntoView(ChatHistory.Items[^1]);
        }

        #endregion

        #region NLP Processing & Activity Log

        private void ProcessUserInput(string input)
        {
            string lower = input.ToLower().Trim();

            // 1. Waiting for user to say "hey"
            if (awaitingGreeting)
            {
                if (lower.Contains("hey"))
                {
                    AppendChat("Bot: Hi! What's your name?");
                    awaitingGreeting = false;
                    awaitingName = true;
                }
                else
                {
                    AppendChat("Bot: Please say 'hey' to start our chat!");
                }
                return;
            }

            // 2. Ask for user name
            if (awaitingName)
            {
                userName = CultureInfo.CurrentCulture.TextInfo.ToTitleCase(input.Trim());
                AppendChat($"Bot: Nice to meet you, {userName}!");
                AppendChat("Bot: What's your favorite cybersecurity topic?");
                awaitingName = false;
                awaitingFavoriteTopic = true;
                return;
            }

            // 3. Ask for favorite topic
            if (awaitingFavoriteTopic)
            {
                favoriteTopic = CultureInfo.CurrentCulture.TextInfo.ToTitleCase(input.Trim());
                AppendChat($"Bot: Awesome! {favoriteTopic} is a great topic.");
                AppendChat("Bot: How are you feeling today?");
                awaitingFavoriteTopic = false;
                awaitingFeeling = true;
                return;
            }

            // 4. Ask for feeling
            if (awaitingFeeling)
            {
                string feeling = input.Trim();
                AppendChat($"Bot: Thanks for sharing that you're feeling '{feeling}', {userName}. I'm here to help you stay safe online!");
                awaitingFeeling = false;
                return;
            }

            // 5. If quiz active, block other commands
            if (quizActive)
            {
                AppendChat("Bot: Quiz is in progress. Please complete the quiz or click 'Next' to proceed.");
                return;
            }

            // Add Task via chat
            if ((lower.Contains("add") || lower.Contains("create")) && lower.Contains("task"))
            {
                string title = Regex.Replace(input, "(?i)add|create|task|to|please|can you|could you|would you|help me|set", "").Trim();
                if (string.IsNullOrEmpty(title))
                    title = "Untitled Task";
                title = CultureInfo.CurrentCulture.TextInfo.ToTitleCase(title);

                var task = new TaskItem { Title = title };
                tasks.Add(task);

                LogAction($"Task added: '{title}' (via chat)");
                AppendChat($"Bot: Task added – '{title}'. Would you like to set a reminder?");
                RefreshTaskList();
                RefreshActivityLog();
                return;
            }

            // Reminder via chat
            if (lower.Contains("remind") || lower.Contains("reminder") || lower.Contains("remember to"))
            {
                var match = Regex.Match(lower, @"remind (me|us)? to (.+?)(?: in (\d+) days?| tomorrow| next day)?$");
                if (!match.Success)
                    match = Regex.Match(lower, @"set reminder (for|to)? (.+?)(?: in (\d+) days?| tomorrow| next day)?$");

                if (match.Success)
                {
                    string action = match.Groups[2].Value.Trim();
                    DateTime? remindDate = null;

                    if (match.Groups.Count > 3 && match.Groups[3].Success && int.TryParse(match.Groups[3].Value, out int days))
                        remindDate = DateTime.Now.Date.AddDays(days);
                    else if (lower.Contains("tomorrow") || lower.Contains("next day"))
                        remindDate = DateTime.Now.Date.AddDays(1);

                    action = CultureInfo.CurrentCulture.TextInfo.ToTitleCase(action);

                    var reminderTask = new TaskItem { Title = action, ReminderDate = remindDate };
                    tasks.Add(reminderTask);

                    LogAction($"Reminder set: '{action}' on {remindDate?.ToString("dd MMM yyyy") ?? "No Date"} (via chat)");
                    AppendChat($"Bot: Reminder set for '{action}' on {(remindDate.HasValue ? remindDate.Value.ToString("dddd, dd MMMM") : "an unspecified date")}.");
                    RefreshTaskList();
                    RefreshActivityLog();
                    return;
                }
            }

            // Show Activity Log via chat
            if (lower.Contains("what have you done") || lower.Contains("summary") || lower.Contains("activity log") ||
                lower.Contains("show log") || lower.Contains("recent actions"))
            {
                if (actionLog.Count == 0)
                {
                    AppendChat("Bot: I haven't performed any actions yet.");
                }
                else
                {
                    AppendChat("Bot: Here's a summary of recent actions:");
                    int start = Math.Max(0, actionLog.Count - LogDisplayLimit);
                    for (int i = start; i < actionLog.Count; i++)
                        AppendChat($"  {i - start + 1}. {actionLog[i]}");
                }
                return;
            }

            // Quiz command via chat
            if (lower.Contains("quiz") || lower.Contains("test") || lower.Contains("challenge") || lower.Contains("exam"))
            {
                AppendChat("Bot: Starting the quiz! Please switch to the 'Cybersecurity Quiz' tab.");
                StartQuiz();
                return;
            }

            // Cybersecurity tips keywords
            if (lower.Contains("phishing") || lower.Contains("password") || lower.Contains("2fa") || lower.Contains("privacy"))
            {
                AppendChat("Bot: That's an important topic! Always verify senders, use unique passwords, and enable 2FA wherever possible.");
                LogAction($"Gave tip on: {input}");
                RefreshActivityLog();
                return;
            }

            // Default fallback
            AppendChat("Bot: I'm not sure I understand. Try asking about tasks, reminders, the quiz, or a cybersecurity topic.");
        }

        private void LogAction(string description)
        {
            string timestamp = DateTime.Now.ToString("HH:mm");
            actionLog.Add($"[{timestamp}] {description}");
            RefreshActivityLog();
        }

        #endregion

        #region Task Management (GUI)

        private void AddTaskButton_Click(object sender, RoutedEventArgs e)
        {
            string title = NewTaskTextBox.Text.Trim();
            if (string.IsNullOrEmpty(title))
            {
                MessageBox.Show("Please enter a task title.", "Validation", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            title = CultureInfo.CurrentCulture.TextInfo.ToTitleCase(title);
            var task = new TaskItem { Title = title };
            tasks.Add(task);

            LogAction($"Task added: '{title}' (via GUI)");
            AppendChat($"Bot: Task added – '{title}'.");
            RefreshTaskList();
            RefreshActivityLog();

            NewTaskTextBox.Clear();
        }

        private void RefreshTaskList()
        {
            TaskListBox.Items.Clear();
            foreach (var task in tasks)
            {
                string display = task.IsCompleted ? $"✔️ {task.Title}" : task.Title;
                if (task.ReminderDate.HasValue)
                    display += $" (Reminder: {task.ReminderDate.Value:dd MMM yyyy})";
                TaskListBox.Items.Add(display);
            }
        }

        private void CompleteTaskButton_Click(object sender, RoutedEventArgs e)
        {
            if (TaskListBox.SelectedIndex < 0)
            {
                MessageBox.Show("Please select a task to mark as completed.", "Validation", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            tasks[TaskListBox.SelectedIndex].IsCompleted = true;
            LogAction($"Task completed: '{tasks[TaskListBox.SelectedIndex].Title}'");
            AppendChat($"Bot: Task '{tasks[TaskListBox.SelectedIndex].Title}' marked as completed.");

            RefreshTaskList();
            RefreshActivityLog();
        }

        #endregion

        #region Activity Log GUI

        private void RefreshActivityLog()
        {
            ActivityLogListBox.Items.Clear();
            int start = Math.Max(0, actionLog.Count - LogDisplayLimit);
            for (int i = start; i < actionLog.Count; i++)
            {
                ActivityLogListBox.Items.Add($"{i - start + 1}. {actionLog[i]}");
            }
        }

        #endregion

        #region Quiz Feature

        private void InitializeQuizQuestions()
        {
            quizQuestions.Clear();

            quizQuestions.Add(new QuizQuestion
            {
                Question = "What should you do if you receive an email asking for your password?",
                Choices = new List<string> { "Reply with password", "Delete the email", "Report the email as phishing", "Ignore it" },
                CorrectAnswer = "Report the email as phishing",
                Explanation = "Correct! Reporting phishing emails helps prevent scams."
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "True or False: Using the same password for multiple accounts is safe.",
                Choices = new List<string> { "True", "False" },
                CorrectAnswer = "False",
                Explanation = "Correct! Using unique passwords for each account helps protect your security.",
                IsTrueFalse = true
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "Which of the following is the safest way to protect your online accounts?",
                Choices = new List<string> { "Using a strong password only", "Using Two-Factor Authentication (2FA)", "Sharing passwords with friends", "Writing passwords on paper" },
                CorrectAnswer = "Using Two-Factor Authentication (2FA)",
                Explanation = "Correct! 2FA adds an extra layer of security beyond just a password."
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "What does 'phishing' mean in cybersecurity?",
                Choices = new List<string> { "Fishing for data", "A type of social engineering attack", "Downloading files", "Encrypting data" },
                CorrectAnswer = "A type of social engineering attack",
                Explanation = "Phishing is a social engineering attack where attackers trick you into giving sensitive info."
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "What is a strong password?",
                Choices = new List<string> { "Your birthdate", "12345678", "A mix of letters, numbers, and symbols", "Your name" },
                CorrectAnswer = "A mix of letters, numbers, and symbols",
                Explanation = "Strong passwords use a combination of letters, numbers, and symbols to increase security."
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "True or False: It's safe to use public Wi-Fi without a VPN for sensitive activities.",
                Choices = new List<string> { "True", "False" },
                CorrectAnswer = "False",
                Explanation = "Using a VPN on public Wi-Fi protects your data from interception."
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "What is Two-Factor Authentication (2FA)?",
                Choices = new List<string> { "Logging in twice", "A security method requiring two forms of ID", "Using two passwords", "Logging out twice" },
                CorrectAnswer = "A security method requiring two forms of ID",
                Explanation = "2FA adds an extra layer of security by requiring two types of verification."
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "What should you do if you suspect your account has been hacked?",
                Choices = new List<string> { "Ignore it", "Change your password immediately", "Post about it on social media", "Delete the account" },
                CorrectAnswer = "Change your password immediately",
                Explanation = "Changing your password immediately helps secure your account."
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "Which one is NOT a common cybersecurity threat?",
                Choices = new List<string> { "Malware", "Phishing", "Firewall", "Ransomware" },
                CorrectAnswer = "Firewall",
                Explanation = "A firewall is a protective measure, not a threat."
            });

            quizQuestions.Add(new QuizQuestion
            {
                Question = "True or False: You should share your passwords with trusted friends.",
                Choices = new List<string> { "True", "False" },
                CorrectAnswer = "False",
                Explanation = "Passwords should never be shared to keep your accounts secure."
            });
        }

        private void StartQuizButton_Click(object sender, RoutedEventArgs e)
        {
            StartQuiz();
        }

        private void StartQuiz()
        {
            if (quizQuestions.Count == 0)
            {
                MessageBox.Show("No quiz questions available.", "Quiz", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            quizActive = true;
            quizScore = 0;
            currentQuizIndex = 0;
            StartQuizButton.IsEnabled = false;
            NextQuestionButton.IsEnabled = false;
            QuizStatusText.Text = $"Question 1 of {quizQuestions.Count}";
            ShowQuestion(quizQuestions[currentQuizIndex]);
            QuizPanel.Visibility = Visibility.Visible;
            AppendChat("Bot: Quiz started. Good luck!");
        }

        private void ShowQuestion(QuizQuestion question)
        {
            QuestionTextBlock.Text = question.Question;
            AnswersPanel.Children.Clear();

            foreach (var choice in question.Choices)
            {
                RadioButton rb = new()
                {
                    Content = choice,
                    Margin = new Thickness(0, 0, 0, 5),
                    GroupName = "QuizAnswers"
                };
                rb.Checked += AnswerSelected;
                AnswersPanel.Children.Add(rb);
            }
        }

        private void AnswerSelected(object sender, RoutedEventArgs e)
        {
            NextQuestionButton.IsEnabled = true;
        }

        private void NextQuestionButton_Click(object sender, RoutedEventArgs e)
        {
            // Check selected answer
            string selectedAnswer = null;
            foreach (RadioButton rb in AnswersPanel.Children)
            {
                if (rb.IsChecked == true)
                {
                    selectedAnswer = rb.Content.ToString();
                    break;
                }
            }

            if (selectedAnswer == null)
            {
                MessageBox.Show("Please select an answer before continuing.", "Quiz", MessageBoxButton.OK, MessageBoxImage.Information);
                return;
            }

            QuizQuestion currentQuestion = quizQuestions[currentQuizIndex];
            if (selectedAnswer.Equals(currentQuestion.CorrectAnswer, StringComparison.OrdinalIgnoreCase))
            {
                quizScore++;
                AppendChat($"Bot: Correct! {currentQuestion.Explanation}");
            }
            else
            {
                AppendChat($"Bot: Incorrect. The correct answer is: {currentQuestion.CorrectAnswer}. {currentQuestion.Explanation}");
            }

            currentQuizIndex++;

            if (currentQuizIndex >= quizQuestions.Count)
            {
                EndQuiz();
            }
            else
            {
                QuizStatusText.Text = $"Question {currentQuizIndex + 1} of {quizQuestions.Count}";
                ShowQuestion(quizQuestions[currentQuizIndex]);
                NextQuestionButton.IsEnabled = false;
            }
        }

        private void EndQuiz()
        {
            quizActive = false;
            StartQuizButton.IsEnabled = true;
            NextQuestionButton.IsEnabled = false;
            QuizPanel.Visibility = Visibility.Collapsed;

            AppendChat($"Bot: Quiz complete! You scored {quizScore} out of {quizQuestions.Count}.");

            string rating;
            if (quizScore == quizQuestions.Count)
                rating = "You're a cybersecurity pro! 🎉";
            else if (quizScore >= quizQuestions.Count / 2)
                rating = "Good job! Keep learning.";
            else
                rating = "Keep practicing and you'll improve!";

            AppendChat("Bot: " + rating);
            LogAction($"Quiz completed with score {quizScore}/{quizQuestions.Count}");
        }

        #endregion
    }
}
