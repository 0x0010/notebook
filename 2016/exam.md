### 第五题

如果定义了星期几的数组，就没必要使用switch了。 看simplest那一行，只要一句就可以。
````java
import java.util.Scanner;

public class Q5 {
    final static String[] weekdays = {"Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"};
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        while (true) {
            System.out.print("Type an integer(1-7) or q(Q) to exit program:");
            String input = scanner.nextLine();
            int inputInt;
            if (input.matches("\\d+") && (inputInt = Integer.parseInt(input)) >= 1 && inputInt <= 7) {
                // switch
                switch (inputInt) {
                    case 1:
                        System.out.println(weekdays[inputInt - 1]);
                        break;
                    case 2:
                        System.out.println(weekdays[inputInt - 1]);
                        break;
                    case 3:
                        System.out.println(weekdays[inputInt - 1]);
                        break;
                    case 4:
                        System.out.println(weekdays[inputInt - 1]);
                        break;
                    case 5:
                        System.out.println(weekdays[inputInt - 1]);
                        break;
                    case 6:
                        System.out.println(weekdays[inputInt - 1]);
                        break;
                    case 7:
                        System.out.println(weekdays[inputInt - 1]);
                        break;
                }
                // simplest
                //System.out.println(weekdays[inputInt-1]);
            } else if (input.equalsIgnoreCase("q")) {
                System.exit(0);
            }
        }
    }
}
````
